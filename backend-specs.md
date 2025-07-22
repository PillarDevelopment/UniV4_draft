# Backend Спецификация: Uniswap V4 Активное Управление Ликвидностью

## Обзор

Backend система обеспечивает критические функции для управления ликвидностью: мониторинг позиций, управление Impermanent Loss, интеграцию с CEX для хеджирования, и координацию с смарт-контрактами.

## Архитектура Системы

### Микросервисная Архитектура

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   API Gateway   │    │  Strategy Core  │    │  CEX Manager    │
│                 │◄──►│                 │◄──►│                 │
│  - Authentication│    │ - IL Management │    │ - Binance API   │
│  - Rate Limiting│    │ - Range Rolling │    │ - Position Sync │
│  - Load Balancing│    │ - Risk Calc     │    │ - Order Exec    │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         ▼                       ▼                       ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│ Blockchain Mon  │    │   Data Engine   │    │  Notification   │
│                 │    │                 │    │                 │
│ - Event Listener│    │ - Price Feeds   │    │ - Alerts        │
│ - Position Track│    │ - Historical    │    │ - Reports       │
│ - Gas Optimizer │    │ - Analytics     │    │ - Dashboards    │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

## Core Сервисы

### 1. Strategy Core Service

**Основной сервис для управления IL и ролирования диапазонов**

#### Компоненты

```typescript
interface ImpermanentLossManager {
  calculateCurrentIL(position: UniswapPosition): ILMetrics;
  calculateOptimalHedge(position: UniswapPosition): HedgeRecommendation;
  executeHedgingStrategy(recommendation: HedgeRecommendation): Promise<HedgeResult>;
  monitorHedgeEffectiveness(hedgeId: string): HedgeEffectiveness;
}

interface RangeRollingManager {
  calculateOptimalRange(pool: PoolData, strategy: StrategyParams): RangeRecommendation;
  shouldRebalance(position: UniswapPosition, market: MarketData): RebalanceDecision;
  executeRangeRoll(position: UniswapPosition, newRange: PriceRange): Promise<RollResult>;
  optimizeForFees(position: UniswapPosition, market: MarketData): FeeOptimization;
}

interface StrategyParams {
  volatilityWindow: number;        // Период для расчета волатильности
  rebalanceThreshold: number;      // Процент приближения к границе
  apyThreshold: number;           // Минимальный APY для ребалансировки
  maxHedgeRatio: number;          // Максимальное отношение хеджа к позиции
  riskTolerance: RiskLevel;       // Уровень риска стратегии
}
```

#### Алгоритм Управления IL

```typescript
class ImpermanentLossManager implements IimpermanentLossManager {
  async calculateCurrentIL(position: UniswapPosition): Promise<ILMetrics> {
    const currentPrice = await this.priceService.getCurrentPrice(position.pool);
    const entryPrice = position.entryPrice;
    const priceRatio = currentPrice / entryPrice;
    
    // Формула расчета IL для концентрированной ликвидности
    const ilPercent = this.calculateConcentratedIL(
      priceRatio,
      position.tickLower,
      position.tickUpper,
      position.entryPrice
    );
    
    const ilUSD = position.liquidityUSD * (ilPercent / 100);
    const feesEarned = await this.calculateAccruedFees(position);
    const netIL = ilUSD - feesEarned;
    
    return {
      ilPercent,
      ilUSD,
      feesEarned,
      netIL,
      priceRatio,
      timestamp: Date.now()
    };
  }
  
  private calculateConcentratedIL(
    priceRatio: number,
    tickLower: number,
    tickUpper: number,
    entryPrice: number
  ): number {
    const priceLower = this.tickToPrice(tickLower);
    const priceUpper = this.tickToPrice(tickUpper);
    const currentPrice = entryPrice * priceRatio;
    
    // Для позиций вне диапазона IL = 100%
    if (currentPrice < priceLower || currentPrice > priceUpper) {
      return 100;
    }
    
    // Расчет IL для позиций в диапазоне
    const sqrtRatio = Math.sqrt(priceRatio);
    const sqrtLower = Math.sqrt(priceLower / entryPrice);
    const sqrtUpper = Math.sqrt(priceUpper / entryPrice);
    
    const liquidityFactor = (sqrtRatio - sqrtLower) / (sqrtUpper - sqrtLower);
    const holdValue = 0.5 * (1 + priceRatio);
    const lpValue = sqrtRatio + liquidityFactor * (sqrtUpper - sqrtRatio);
    
    return ((holdValue - lpValue) / holdValue) * 100;
  }
  
  async calculateOptimalHedge(position: UniswapPosition): Promise<HedgeRecommendation> {
    const ilMetrics = await this.calculateCurrentIL(position);
    const volatility = await this.marketAnalyzer.getHistoricalVolatility(
      position.pool,
      this.strategyParams.volatilityWindow
    );
    
    // Расчет дельты позиции
    const delta = this.calculatePositionDelta(position);
    
    // Определение оптимального размера хеджа
    const hedgeSize = Math.min(
      Math.abs(delta) * position.liquidityUSD,
      position.liquidityUSD * this.strategyParams.maxHedgeRatio
    );
    
    // Выбор инструмента хеджирования
    const hedgeInstrument = await this.selectHedgeInstrument(
      position.pool.token,
      hedgeSize,
      volatility
    );
    
    return {
      action: delta > 0 ? 'SHORT' : 'LONG',
      size: hedgeSize,
      instrument: hedgeInstrument,
      expectedILReduction: this.estimateILReduction(hedgeSize, delta),
      cost: await this.estimateHedgeCost(hedgeInstrument, hedgeSize),
      confidence: this.calculateConfidence(volatility, ilMetrics)
    };
  }
}
```

#### Алгоритм Ролирования Диапазонов

```typescript
class RangeRollingManager implements IRangeRollingManager {
  async calculateOptimalRange(
    pool: PoolData,
    strategy: StrategyParams
  ): Promise<RangeRecommendation> {
    const currentPrice = pool.currentPrice;
    const volatility = await this.getVolatility(pool, strategy.volatilityWindow);
    const liquidityDistribution = await this.getLiquidityDistribution(pool);
    
    // Расчет оптимальной ширины диапазона на основе волатильности
    const rangeWidth = this.calculateOptimalWidth(volatility, strategy.riskTolerance);
    
    // Определение асимметрии диапазона на основе рыночных условий
    const asymmetryFactor = await this.calculateAsymmetry(pool, liquidityDistribution);
    
    const tickSpacing = pool.tickSpacing;
    const currentTick = this.priceToTick(currentPrice);
    
    // Расчет новых границ
    const tickRange = Math.round(rangeWidth / tickSpacing) * tickSpacing;
    const lowerOffset = Math.round(tickRange * (0.5 + asymmetryFactor));
    const upperOffset = tickRange - lowerOffset;
    
    const newTickLower = currentTick - lowerOffset;
    const newTickUpper = currentTick + upperOffset;
    
    // Оптимизация для максимизации fees
    const optimizedRange = await this.optimizeForFees({
      tickLower: newTickLower,
      tickUpper: newTickUpper,
      currentTick,
      liquidityDistribution
    });
    
    return {
      tickLower: optimizedRange.tickLower,
      tickUpper: optimizedRange.tickUpper,
      expectedAPY: await this.estimateAPY(optimizedRange, pool),
      confidence: this.calculateRangeConfidence(volatility, liquidityDistribution),
      reasoning: this.generateReasoning(optimizedRange, pool, strategy)
    };
  }
  
  async shouldRebalance(
    position: UniswapPosition,
    market: MarketData
  ): Promise<RebalanceDecision> {
    const currentTick = this.priceToTick(market.currentPrice);
    const distanceToLower = currentTick - position.tickLower;
    const distanceToUpper = position.tickUpper - currentTick;
    const rangeWidth = position.tickUpper - position.tickLower;
    
    // Проверка ценовых триггеров
    const lowerThreshold = rangeWidth * (this.strategyParams.rebalanceThreshold / 100);
    const upperThreshold = rangeWidth * (this.strategyParams.rebalanceThreshold / 100);
    
    const nearLowerBound = distanceToLower <= lowerThreshold;
    const nearUpperBound = distanceToUpper <= upperThreshold;
    
    // Проверка APY триггеров
    const currentAPY = await this.calculateCurrentAPY(position);
    const belowAPYThreshold = currentAPY < this.strategyParams.apyThreshold;
    
    // Проверка эффективности капитала
    const capitalEfficiency = await this.calculateCapitalEfficiency(position);
    const inefficientCapital = capitalEfficiency < 0.7; // 70% порог
    
    const shouldRebalance = nearLowerBound || nearUpperBound || 
                           belowAPYThreshold || inefficientCapital;
    
    return {
      shouldRebalance,
      triggers: {
        priceProximity: nearLowerBound || nearUpperBound,
        lowAPY: belowAPYThreshold,
        lowEfficiency: inefficientCapital
      },
      urgency: this.calculateUrgency(distanceToLower, distanceToUpper, currentAPY),
      estimatedGasCost: await this.estimateRebalanceGasCost(position),
      expectedImprovement: await this.estimateImprovementFromRebalance(position, market)
    };
  }
  
  private calculateOptimalWidth(volatility: number, riskTolerance: RiskLevel): number {
    // Базовая ширина в зависимости от волатильности
    const baseWidth = volatility * 2; // 2 стандартных отклонения
    
    // Корректировка на основе риск-толерантности
    const riskMultipliers = {
      [RiskLevel.Conservative]: 1.5,
      [RiskLevel.Moderate]: 1.0,
      [RiskLevel.Aggressive]: 0.7
    };
    
    return baseWidth * riskMultipliers[riskTolerance];
  }
  
  private async optimizeForFees(rangeParams: any): Promise<OptimizedRange> {
    // Анализ распределения ликвидности для поиска "fee-rich" зон
    const liquidityGaps = this.findLiquidityGaps(rangeParams.liquidityDistribution);
    const feeHotspots = await this.identifyFeeHotspots(rangeParams);
    
    // Корректировка диапазона для захвата максимальных fees
    let optimizedLower = rangeParams.tickLower;
    let optimizedUpper = rangeParams.tickUpper;
    
    // Смещение к зонам с высокой торговой активностью
    if (feeHotspots.length > 0) {
      const hotspotCenter = feeHotspots.reduce((sum, spot) => sum + spot.tick, 0) / feeHotspots.length;
      const currentCenter = (rangeParams.tickLower + rangeParams.tickUpper) / 2;
      const shift = (hotspotCenter - currentCenter) * 0.3; // 30% смещение к hotspot
      
      optimizedLower += Math.round(shift);
      optimizedUpper += Math.round(shift);
    }
    
    return {
      tickLower: optimizedLower,
      tickUpper: optimizedUpper,
      expectedFeeMultiplier: this.calculateFeeMultiplier(liquidityGaps, feeHotspots)
    };
  }
}
```

### 2. CEX Manager Service

**Сервис для интеграции с централизованными биржами и управления хеджированием**

```typescript
interface CEXManager {
  executeHedge(hedge: HedgeRecommendation): Promise<HedgeExecution>;
  monitorPositions(): Promise<CEXPosition[]>;
  syncPositions(): Promise<SyncResult>;
  calculateRequiredMargin(positions: CEXPosition[]): Promise<MarginRequirement>;
}

class CEXManager implements ICEXManager {
  private exchanges: Map<string, ExchangeConnector> = new Map();
  private positionTracker: CEXPositionTracker;
  
  constructor(
    private apiKeyManager: APIKeyManager,
    private riskManager: RiskManager
  ) {
    this.initializeExchanges();
  }
  
  async executeHedge(hedge: HedgeRecommendation): Promise<HedgeExecution> {
    const exchange = this.selectOptimalExchange(hedge);
    const account = await this.getAccountInfo(exchange);
    
    // Проверка доступной маржи
    const requiredMargin = await this.calculateRequiredMargin([hedge]);
    if (account.availableMargin < requiredMargin.total) {
      throw new Error('Insufficient margin for hedge execution');
    }
    
    // Исполнение сделки
    const order = await exchange.createOrder({
      symbol: hedge.instrument.symbol,
      type: 'market',
      side: hedge.action.toLowerCase(),
      amount: hedge.size,
      params: {
        leverage: hedge.instrument.leverage || 1
      }
    });
    
    // Трекинг исполненной позиции
    const position = await this.trackNewPosition(order, hedge);
    
    return {
      orderId: order.id,
      executedPrice: order.price,
      executedSize: order.filled,
      fees: order.fee,
      position: position,
      timestamp: Date.now()
    };
  }
  
  async syncPositions(): Promise<SyncResult> {
    const allPositions = await this.getAllCEXPositions();
    const uniswapPositions = await this.getUniswapPositions();
    
    const syncAnalysis = this.analyzeSynchronization(allPositions, uniswapPositions);
    
    if (syncAnalysis.requiresAction) {
      await this.executeSync Actions(syncAnalysis.actions);
    }
    
    return {
      totalPositions: allPositions.length,
      syncedPositions: syncAnalysis.syncedCount,
      discrepancies: syncAnalysis.discrepancies,
      actionsExecuted: syncAnalysis.actions.length,
      timestamp: Date.now()
    };
  }
  
  private async analyzeSynchronization(
    cexPositions: CEXPosition[],
    uniswapPositions: UniswapPosition[]
  ): Promise<SyncAnalysis> {
    const discrepancies: PositionDiscrepancy[] = [];
    const actions: SyncAction[] = [];
    
    for (const uniPos of uniswapPositions) {
      const requiredHedge = await this.calculateRequiredHedge(uniPos);
      const currentCEXExposure = this.calculateCurrentCEXExposure(uniPos.pool.token, cexPositions);
      
      const exposureDiff = requiredHedge.size - currentCEXExposure.size;
      const toleranceThreshold = requiredHedge.size * 0.05; // 5% tolerance
      
      if (Math.abs(exposureDiff) > toleranceThreshold) {
        discrepancies.push({
          pool: uniPos.pool,
          requiredExposure: requiredHedge.size,
          currentExposure: currentCEXExposure.size,
          difference: exposureDiff,
          severity: this.calculateDiscrepancySeverity(exposureDiff, requiredHedge.size)
        });
        
        actions.push({
          type: exposureDiff > 0 ? 'INCREASE_HEDGE' : 'DECREASE_HEDGE',
          size: Math.abs(exposureDiff),
          instrument: requiredHedge.instrument,
          priority: discrepancies[discrepancies.length - 1].severity
        });
      }
    }
    
    return {
      requiresAction: discrepancies.length > 0,
      discrepancies,
      actions,
      syncedCount: uniswapPositions.length - discrepancies.length
    };
  }
}
```

### 3. Blockchain Monitor Service

**Сервис для мониторинга блокчейна и событий смарт-контрактов**

```typescript
class BlockchainMonitor {
  private eventListeners: Map<string, EventListener> = new Map();
  private positionCache: Map<string, UniswapPosition> = new Map();
  
  async startMonitoring(): Promise<void> {
    // Мониторинг событий vault контракта
    this.setupVaultEventListeners();
    
    // Мониторинг Uniswap V4 событий
    this.setupUniswapEventListeners();
    
    // Мониторинг цен и рыночных данных
    this.setupPriceMonitoring();
    
    // Мониторинг газовых цен для оптимизации транзакций
    this.setupGasMonitoring();
  }
  
  private setupVaultEventListeners(): void {
    const vaultContract = this.getVaultContract();
    
    // Мониторинг депозитов
    vaultContract.on('DepositMade', async (user, assets, shares, event) => {
      await this.handleDepositEvent({
        user,
        assets: assets.toString(),
        shares: shares.toString(),
        blockNumber: event.blockNumber,
        transactionHash: event.transactionHash
      });
    });
    
    // Мониторинг выводов
    vaultContract.on('WithdrawalExecuted', async (user, assets, event) => {
      await this.handleWithdrawalEvent({
        user,
        assets: assets.toString(),
        blockNumber: event.blockNumber,
        transactionHash: event.transactionHash
      });
    });
    
    // Мониторинг сбора комиссий
    vaultContract.on('FeesCollected', async (managementFee, performanceFee, event) => {
      await this.handleFeesCollectedEvent({
        managementFee: managementFee.toString(),
        performanceFee: performanceFee.toString(),
        totalFees: managementFee.add(performanceFee).toString(),
        blockNumber: event.blockNumber
      });
    });
  }
  
  private setupUniswapEventListeners(): void {
    const hookContract = this.getHookContract();
    
    // Мониторинг создания позиций
    hookContract.on('PositionCreated', async (poolKey, tickLower, tickUpper, liquidity, event) => {
      const position = await this.buildPositionFromEvent({
        poolKey,
        tickLower: tickLower.toString(),
        tickUpper: tickUpper.toString(),
        liquidity: liquidity.toString(),
        blockNumber: event.blockNumber
      });
      
      await this.notifyStrategyCore('POSITION_CREATED', position);
    });
    
    // Мониторинг ребалансировок
    hookContract.on('PositionRebalanced', async (...args) => {
      await this.handleRebalanceEvent(args);
    });
    
    // Мониторинг сбора fees
    hookContract.on('FeesCollected', async (...args) => {
      await this.handleUniswapFeesEvent(args);
    });
  }
  
  async handleDepositEvent(eventData: DepositEventData): Promise<void> {
    // Обновление внутренних записей
    await this.updateUserBalance(eventData.user, eventData.shares);
    
    // Уведомление Strategy Core о новых средствах для инвестирования
    await this.strategyCore.notifyNewCapital({
      amount: eventData.assets,
      timestamp: Date.now()
    });
    
    // Отправка уведомления пользователю
    await this.notificationService.sendDepositConfirmation(eventData);
    
    // Логирование для аналитики
    await this.analytics.recordDeposit(eventData);
  }
}
```

### 4. Data Engine Service

**Сервис для управления данными, аналитикой и историческими записями**

```typescript
class DataEngine {
  private priceProviders: PriceProvider[] = [];
  private analyticsDB: AnalyticsDatabase;
  private metricsCalculator: MetricsCalculator;
  
  async getHistoricalVolatility(
    token: string,
    windowDays: number
  ): Promise<VolatilityData> {
    const prices = await this.getPriceHistory(token, windowDays);
    const returns = this.calculateReturns(prices);
    
    return {
      annualizedVolatility: this.calculateAnnualizedVolatility(returns),
      dailyVolatility: this.calculateDailyVolatility(returns),
      realizedVariance: this.calculateRealizedVariance(returns),
      skewness: this.calculateSkewness(returns),
      kurtosis: this.calculateKurtosis(returns),
      confidenceInterval: this.calculateConfidenceInterval(returns)
    };
  }
  
  async calculateStrategyPerformance(
    strategyId: string,
    period: TimePeriod
  ): Promise<PerformanceMetrics> {
    const positions = await this.getStrategyPositions(strategyId, period);
    const trades = await this.getStrategyTrades(strategyId, period);
    
    return {
      totalReturn: this.calculateTotalReturn(positions, trades),
      sharpeRatio: this.calculateSharpeRatio(positions, trades),
      maxDrawdown: this.calculateMaxDrawdown(positions),
      winRate: this.calculateWinRate(trades),
      avgTradeDuration: this.calculateAvgTradeDuration(trades),
      avgPnLPerTrade: this.calculateAvgPnL(trades),
      volatility: this.calculateStrategyVolatility(positions),
      calmarRatio: this.calculateCalmarRatio(positions, trades),
      sortinoRatio: this.calculateSortinoRatio(positions, trades)
    };
  }
  
  async performABTest(
    strategyA: StrategyConfig,
    strategyB: StrategyConfig,
    testDuration: number
  ): Promise<ABTestResult> {
    const testId = this.generateTestId();
    
    // Запуск параллельных стратегий
    const [resultA, resultB] = await Promise.all([
      this.backtest(strategyA, testDuration),
      this.backtest(strategyB, testDuration)
    ]);
    
    // Статистический анализ результатов
    const significance = this.calculateStatisticalSignificance(resultA, resultB);
    const confidenceInterval = this.calculateConfidenceInterval(resultA, resultB);
    
    return {
      testId,
      strategyA: {
        config: strategyA,
        performance: resultA,
        metrics: await this.calculateStrategyPerformance(testId + '_A', testDuration)
      },
      strategyB: {
        config: strategyB,
        performance: resultB,
        metrics: await this.calculateStrategyPerformance(testId + '_B', testDuration)
      },
      statisticalSignificance: significance,
      confidenceInterval,
      recommendation: this.generateRecommendation(resultA, resultB, significance),
      timestamp: Date.now()
    };
  }
}
```

### 5. API Gateway

**Главный API для взаимодействия с фронтендом и внешними системами**

```typescript
@Controller('/api/v1')
class APIController {
  constructor(
    private strategyCore: StrategyCore,
    private cexManager: CEXManager,
    private dataEngine: DataEngine,
    private authService: AuthService
  ) {}
  
  @Get('/vault/status')
  @UseGuards(AuthGuard)
  async getVaultStatus(): Promise<VaultStatus> {
    const totalAssets = await this.strategyCore.getTotalAssets();
    const positions = await this.strategyCore.getActivePositions();
    const performance = await this.dataEngine.getVaultPerformance();
    
    return {
      totalValueLocked: totalAssets,
      activePositions: positions.length,
      currentAPY: performance.annualizedReturn,
      sharpeRatio: performance.sharpeRatio,
      maxDrawdown: performance.maxDrawdown,
      lastUpdate: Date.now()
    };
  }
  
  @Get('/user/:address/portfolio')
  @UseGuards(AuthGuard)
  async getUserPortfolio(@Param('address') address: string): Promise<UserPortfolio> {
    const userShares = await this.strategyCore.getUserShares(address);
    const totalShares = await this.strategyCore.getTotalShares();
    const vaultValue = await this.strategyCore.getTotalAssets();
    
    const userValue = (userShares / totalShares) * vaultValue;
    const performance = await this.dataEngine.getUserPerformance(address);
    
    return {
      balance: {
        shares: userShares,
        valueUSD: userValue,
        percentage: (userShares / totalShares) * 100
      },
      performance: {
        totalReturn: performance.totalReturn,
        apy: performance.currentAPY,
        since: performance.entryDate
      },
      nextWithdrawalWindow: await this.getNextWithdrawalWindow()
    };
  }
  
  @Post('/strategy/rebalance')
  @UseGuards(StrategyManagerGuard)
  async triggerRebalance(@Body() rebalanceRequest: RebalanceRequest): Promise<RebalanceResult> {
    // Валидация запроса
    const validation = await this.validateRebalanceRequest(rebalanceRequest);
    if (!validation.isValid) {
      throw new BadRequestException(validation.errors);
    }
    
    // Выполнение ребалансировки
    const result = await this.strategyCore.executeRebalance(rebalanceRequest);
    
    // Логирование для аудита
    await this.auditService.logRebalance({
      request: rebalanceRequest,
      result,
      executor: rebalanceRequest.executorAddress,
      timestamp: Date.now()
    });
    
    return result;
  }
  
  @Get('/analytics/strategy-comparison')
  @UseGuards(AuthGuard)
  async getStrategyComparison(@Query() params: ComparisonParams): Promise<StrategyComparison> {
    const strategies = await this.dataEngine.getStrategies(params.strategyIds);
    const comparison = await this.dataEngine.compareStrategies(strategies, params.period);
    
    return {
      strategies: comparison.strategies,
      metrics: comparison.metrics,
      recommendation: comparison.bestPerformer,
      period: params.period,
      generatedAt: Date.now()
    };
  }
}
```

## Развертывание и Конфигурация

### Docker Configuration

```yaml
# docker-compose.yml
version: '3.8'

services:
  api-gateway:
    build: ./services/api-gateway
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - DATABASE_URL=${DATABASE_URL}
      - REDIS_URL=${REDIS_URL}
    depends_on:
      - postgres
      - redis
      
  strategy-core:
    build: ./services/strategy-core
    environment:
      - NODE_ENV=production
      - BLOCKCHAIN_RPC=${RPC_URL}
      - CEX_API_KEYS_ENCRYPTED=${ENCRYPTED_KEYS}
    depends_on:
      - postgres
      - redis
      
  cex-manager:
    build: ./services/cex-manager
    environment:
      - BINANCE_API_KEY=${BINANCE_API_KEY}
      - BINANCE_SECRET=${BINANCE_SECRET}
      - BYBIT_API_KEY=${BYBIT_API_KEY}
      - BYBIT_SECRET=${BYBIT_SECRET}
    restart: always
    
  blockchain-monitor:
    build: ./services/blockchain-monitor
    environment:
      - RPC_URLS=${RPC_URLS}
      - CONTRACT_ADDRESSES=${CONTRACT_ADDRESSES}
    restart: always
    
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: liquidity_vault
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      
  redis:
    image: redis:7
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  redis_data:
```

### Environment Configuration

```typescript
// config/environment.ts
export const config = {
  database: {
    url: process.env.DATABASE_URL,
    maxConnections: 10,
    ssl: process.env.NODE_ENV === 'production'
  },
  blockchain: {
    rpcUrl: process.env.RPC_URL,
    contracts: {
      vault: process.env.VAULT_CONTRACT_ADDRESS,
      hook: process.env.HOOK_CONTRACT_ADDRESS,
      strategyManager: process.env.STRATEGY_MANAGER_ADDRESS
    },
    gasOptimization: {
      maxGasPrice: '100', // gwei
      priorityFee: '2'    // gwei
    }
  },
  cex: {
    exchanges: {
      binance: {
        apiKey: process.env.BINANCE_API_KEY,
        secret: process.env.BINANCE_SECRET,
        sandbox: process.env.NODE_ENV !== 'production'
      },
      bybit: {
        apiKey: process.env.BYBIT_API_KEY,
        secret: process.env.BYBIT_SECRET,
        testnet: process.env.NODE_ENV !== 'production'
      }
    },
    defaultLeverage: 3,
    maxPositionSize: 1000000, // $1M
    riskLimits: {
      maxDrawdown: 0.15,      // 15%
      maxExposure: 0.8        // 80% of vault
    }
  },
  strategy: {
    rebalanceThresholds: {
      priceProximity: 0.05,   // 5%
      apyThreshold: 0.10,     // 10%
      capitalEfficiency: 0.70  // 70%
    },
    riskManagement: {
      maxIL: 0.20,            // 20%
      hedgeRatio: 0.80,       // 80%
      stopLoss: 0.25          // 25%
    }
  }
};
```

## Мониторинг и Алерты

### Система Мониторинга

```typescript
class MonitoringSystem {
  private alertManager: AlertManager;
  private metricsCollector: MetricsCollector;
  
  async initializeMonitoring(): Promise<void> {
    // Мониторинг производительности системы
    this.setupPerformanceMonitoring();
    
    // Мониторинг финансовых метрик
    this.setupFinancialMonitoring();
    
    // Мониторинг рисков
    this.setupRiskMonitoring();
    
    // Мониторинг внешних API
    this.setupExternalAPIMonitoring();
  }
  
  private setupRiskMonitoring(): void {
    // Алерт при превышении максимальной просадки
    this.alertManager.addAlert({
      name: 'MaxDrawdownExceeded',
      condition: async () => {
        const currentDrawdown = await this.calculateCurrentDrawdown();
        return currentDrawdown > config.cex.riskLimits.maxDrawdown;
      },
      severity: AlertSeverity.CRITICAL,
      action: async () => {
        await this.strategyCore.pauseTrading();
        await this.notificationService.notifyEmergency('Max drawdown exceeded');
      }
    });
    
    // Алерт при рассинхронизации позиций
    this.alertManager.addAlert({
      name: 'PositionSyncError',
      condition: async () => {
        const syncResult = await this.cexManager.checkSynchronization();
        return syncResult.discrepancies.length > 0;
      },
      severity: AlertSeverity.HIGH,
      action: async () => {
        await this.cexManager.forceSynchronization();
      }
    });
  }
}
```

## Безопасность

### API Key Management

```typescript
class APIKeyManager {
  private encryptionService: EncryptionService;
  
  async storeAPIKey(exchange: string, apiKey: string, secret: string): Promise<void> {
    const encryptedKey = await this.encryptionService.encrypt(apiKey);
    const encryptedSecret = await this.encryptionService.encrypt(secret);
    
    await this.database.storeCredentials({
      exchange,
      apiKey: encryptedKey,
      secret: encryptedSecret,
      permissions: ['trade'], // Только торговые права, без вывода
      createdAt: Date.now()
    });
  }
  
  async rotateAPIKeys(): Promise<void> {
    const exchanges = await this.getActiveExchanges();
    
    for (const exchange of exchanges) {
      try {
        // Создание новых API ключей
        const newCredentials = await exchange.createAPIKey({
          permissions: ['trade'],
          ipRestrictions: [process.env.SERVER_IP]
        });
        
        // Обновление в базе данных
        await this.storeAPIKey(exchange.name, newCredentials.key, newCredentials.secret);
        
        // Деактивация старых ключей
        await this.deactivateOldKeys(exchange.name);
        
      } catch (error) {
        await this.alertManager.sendAlert({
          type: 'API_KEY_ROTATION_FAILED',
          exchange: exchange.name,
          error: error.message
        });
      }
    }
  }
}
```

Это завершает спецификацию backend системы. Документ покрывает все критические компоненты для управления IL, ролирования диапазонов, интеграции с CEX и мониторинга системы.