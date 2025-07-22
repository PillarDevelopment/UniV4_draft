# Smart Contract Спецификация: Uniswap V4 Активное Управление Ликвидностью

## Обзор

Система смарт-контрактов для управления ликвидностью на Uniswap V4 с автоматическим ребалансом и интеграцией с внешними хеджирующими стратегиями.

## Архитектура Контрактов

### 1. ActiveLiquidityVault (ERC-4626)

**Основной контракт для управления депозитами инвесторов**

#### Основные Функции

```solidity
contract ActiveLiquidityVault is ERC4626, Ownable, ReentrancyGuard {
    // Структуры данных
    struct WithdrawalWindow {
        uint256 startTime;
        uint256 endTime;
        bool active;
    }
    
    struct PerformanceFee {
        uint256 managementFeeRate; // 1.5% annually = 150 basis points
        uint256 performanceFeeRate; // 20% = 2000 basis points
        uint256 lastFeeCollection;
        uint256 highWaterMark;
    }
    
    // Состояние контракта
    mapping(address => uint256) public userShares;
    WithdrawalWindow public currentWithdrawalWindow;
    PerformanceFee public feeStructure;
    address public strategyManager;
    address public uniswapHook;
    uint256 public constant MIN_DEPOSIT = 100_000 * 1e6; // $100,000 USDC
    
    // События
    event DepositMade(address indexed user, uint256 assets, uint256 shares);
    event WithdrawalRequested(address indexed user, uint256 shares);
    event WithdrawalExecuted(address indexed user, uint256 assets);
    event FeesCollected(uint256 managementFee, uint256 performanceFee);
    event WithdrawalWindowOpened(uint256 startTime, uint256 endTime);
}
```

#### Ключевые Методы

**Депозиты:**
```solidity
function deposit(uint256 assets, address receiver) 
    external 
    override 
    nonReentrant 
    returns (uint256 shares) 
{
    require(assets >= MIN_DEPOSIT, "Minimum deposit not met");
    require(receiver != address(0), "Invalid receiver");
    
    // Расчет количества shares на основе текущего NAV
    shares = previewDeposit(assets);
    
    // Перевод активов
    IERC20(asset()).safeTransferFrom(msg.sender, address(this), assets);
    
    // Минт shares
    _mint(receiver, shares);
    
    emit DepositMade(receiver, assets, shares);
    
    return shares;
}
```

**Выводы (еженедельное окно):**
```solidity
function requestWithdrawal(uint256 shares) external {
    require(shares > 0, "Invalid shares amount");
    require(balanceOf(msg.sender) >= shares, "Insufficient shares");
    require(currentWithdrawalWindow.active, "Withdrawal window closed");
    require(block.timestamp >= currentWithdrawalWindow.startTime &&
            block.timestamp <= currentWithdrawalWindow.endTime, "Outside withdrawal window");
    
    // Сжигание shares и расчет активов для вывода
    uint256 assets = previewRedeem(shares);
    _burn(msg.sender, shares);
    
    // Перевод активов пользователю
    IERC20(asset()).safeTransfer(msg.sender, assets);
    
    emit WithdrawalExecuted(msg.sender, assets);
}

function openWithdrawalWindow() external onlyOwner {
    currentWithdrawalWindow = WithdrawalWindow({
        startTime: block.timestamp,
        endTime: block.timestamp + 24 hours, // 24-часовое окно каждую неделю
        active: true
    });
    
    emit WithdrawalWindowOpened(currentWithdrawalWindow.startTime, currentWithdrawalWindow.endTime);
}
```

**Расчет NAV и комиссий:**
```solidity
function totalAssets() public view override returns (uint256) {
    // Общая стоимость активов = активы в контракте + стоимость позиций в Uniswap
    uint256 contractBalance = IERC20(asset()).balanceOf(address(this));
    uint256 uniswapPositionValue = IUniswapHook(uniswapHook).getPositionValue();
    
    return contractBalance + uniswapPositionValue;
}

function collectFees() external onlyOwner {
    uint256 currentNav = totalAssets();
    uint256 timeElapsed = block.timestamp - feeStructure.lastFeeCollection;
    
    // Management fee (1.5% annually)
    uint256 managementFee = (currentNav * feeStructure.managementFeeRate * timeElapsed) / 
                           (10000 * 365 days);
    
    // Performance fee (20% от прибыли выше high water mark)
    uint256 performanceFee = 0;
    if (currentNav > feeStructure.highWaterMark) {
        uint256 profit = currentNav - feeStructure.highWaterMark;
        performanceFee = (profit * feeStructure.performanceFeeRate) / 10000;
        feeStructure.highWaterMark = currentNav;
    }
    
    uint256 totalFees = managementFee + performanceFee;
    if (totalFees > 0) {
        // Минт shares для команды как комиссия
        uint256 feeShares = convertToShares(totalFees);
        _mint(owner(), feeShares);
    }
    
    feeStructure.lastFeeCollection = block.timestamp;
    emit FeesCollected(managementFee, performanceFee);
}
```

### 2. UniswapV4LiquidityHook

**Hook-контракт для интеграции с Uniswap V4**

#### Структура Hook

```solidity
contract UniswapV4LiquidityHook is BaseHook {
    using TickMath for int24;
    using LiquidityAmounts for uint256;
    
    struct PositionInfo {
        uint128 liquidity;
        int24 tickLower;
        int24 tickUpper;
        uint256 token0Amount;
        uint256 token1Amount;
        uint256 lastRebalance;
        bool active;
    }
    
    struct RebalanceParams {
        int24 newTickLower;
        int24 newTickUpper;
        uint256 priceThreshold;
        uint256 apyThreshold;
    }
    
    // Состояние
    mapping(PoolKey => PositionInfo) public positions;
    mapping(PoolKey => RebalanceParams) public rebalanceParams;
    address public vault;
    address public strategyManager;
    
    // События
    event PositionCreated(PoolKey indexed poolKey, int24 tickLower, int24 tickUpper, uint128 liquidity);
    event PositionRebalanced(PoolKey indexed poolKey, int24 oldTickLower, int24 oldTickUpper, 
                           int24 newTickLower, int24 newTickUpper);
    event FeesCollected(PoolKey indexed poolKey, uint256 amount0, uint256 amount1);
}
```

#### Основные Hook Функции

**После обмена (AfterSwap):**
```solidity
function afterSwap(
    address sender,
    PoolKey calldata key,
    IPoolManager.SwapParams calldata params,
    BalanceDelta delta,
    bytes calldata hookData
) external override returns (bytes4) {
    PositionInfo storage position = positions[key];
    if (!position.active) return BaseHook.afterSwap.selector;
    
    // Получение текущей цены
    (uint160 sqrtPriceX96,,,,) = poolManager.getSlot0(key.toId());
    int24 currentTick = TickMath.getTickAtSqrtRatio(sqrtPriceX96);
    
    // Проверка необходимости ребалансировки по ценовым триггерам
    RebalanceParams memory rebalanceConfig = rebalanceParams[key];
    
    bool needsRebalance = currentTick <= position.tickLower + rebalanceConfig.priceThreshold ||
                         currentTick >= position.tickUpper - rebalanceConfig.priceThreshold;
    
    if (needsRebalance) {
        _triggerRebalance(key);
    }
    
    return BaseHook.afterSwap.selector;
}
```

**Управление позициями:**
```solidity
function createPosition(
    PoolKey calldata key,
    int24 tickLower,
    int24 tickUpper,
    uint256 amount0Desired,
    uint256 amount1Desired
) external onlyVault returns (uint128 liquidity) {
    // Расчет количества ликвидности
    (uint160 sqrtPriceX96,,,,) = poolManager.getSlot0(key.toId());
    
    liquidity = LiquidityAmounts.getLiquidityForAmounts(
        sqrtPriceX96,
        TickMath.getSqrtRatioAtTick(tickLower),
        TickMath.getSqrtRatioAtTick(tickUpper),
        amount0Desired,
        amount1Desired
    );
    
    // Добавление ликвидности в пул
    BalanceDelta delta = poolManager.modifyLiquidity(
        key,
        IPoolManager.ModifyLiquidityParams({
            tickLower: tickLower,
            tickUpper: tickUpper,
            liquidityDelta: int256(int128(liquidity))
        }),
        ""
    );
    
    // Сохранение информации о позиции
    positions[key] = PositionInfo({
        liquidity: liquidity,
        tickLower: tickLower,
        tickUpper: tickUpper,
        token0Amount: uint256(uint128(-delta.amount0())),
        token1Amount: uint256(uint128(-delta.amount1())),
        lastRebalance: block.timestamp,
        active: true
    });
    
    emit PositionCreated(key, tickLower, tickUpper, liquidity);
    return liquidity;
}

function rebalancePosition(
    PoolKey calldata key,
    int24 newTickLower,
    int24 newTickUpper
) external onlyStrategyManager {
    PositionInfo storage position = positions[key];
    require(position.active, "Position not active");
    
    // Закрытие старой позиции
    poolManager.modifyLiquidity(
        key,
        IPoolManager.ModifyLiquidityParams({
            tickLower: position.tickLower,
            tickUpper: position.tickUpper,
            liquidityDelta: -int256(int128(position.liquidity))
        }),
        ""
    );
    
    // Сбор accumulated fees
    _collectFees(key);
    
    // Открытие новой позиции с новыми параметрами
    // (логика аналогичная createPosition)
    
    emit PositionRebalanced(key, position.tickLower, position.tickUpper, newTickLower, newTickUpper);
}
```

**Сбор комиссий:**
```solidity
function collectFees(PoolKey calldata key) external returns (uint256 amount0, uint256 amount1) {
    PositionInfo storage position = positions[key];
    require(position.active, "Position not active");
    
    // Сбор accumulated fees из позиции
    BalanceDelta delta = poolManager.modifyLiquidity(
        key,
        IPoolManager.ModifyLiquidityParams({
            tickLower: position.tickLower,
            tickUpper: position.tickUpper,
            liquidityDelta: 0 // Нулевое изменение ликвидности для сбора fees
        }),
        ""
    );
    
    amount0 = uint256(uint128(delta.amount0()));
    amount1 = uint256(uint128(delta.amount1()));
    
    // Перевод собранных комиссий в vault
    if (amount0 > 0) IERC20(Currency.unwrap(key.currency0)).safeTransfer(vault, amount0);
    if (amount1 > 0) IERC20(Currency.unwrap(key.currency1)).safeTransfer(vault, amount1);
    
    emit FeesCollected(key, amount0, amount1);
    return (amount0, amount1);
}
```

### 3. StrategyManager

**Контракт для управления стратегиями и координации с бэкендом**

```solidity
contract StrategyManager is Ownable, Pausable {
    struct StrategyParams {
        uint256 rebalanceThreshold; // Процент приближения к границе для ребалансировки
        uint256 apyThreshold;       // Минимальный APY для ребалансировки
        uint256 maxSlippage;        // Максимальный slippage при ребалансировке
        bool autoRebalanceEnabled;
    }
    
    address public vault;
    address public uniswapHook;
    address public backendOperator; // Адрес для взаимодействия с бэкендом
    
    mapping(PoolKey => StrategyParams) public strategyParams;
    
    // События
    event StrategyUpdated(PoolKey indexed poolKey, StrategyParams params);
    event RebalanceTriggered(PoolKey indexed poolKey, string reason);
    event BackendOperatorUpdated(address oldOperator, address newOperator);
    
    modifier onlyBackend() {
        require(msg.sender == backendOperator, "Only backend operator");
        _;
    }
    
    // Функции для бэкенда
    function triggerRebalance(
        PoolKey calldata key,
        int24 newTickLower,
        int24 newTickUpper,
        string calldata reason
    ) external onlyBackend whenNotPaused {
        IUniswapHook(uniswapHook).rebalancePosition(key, newTickLower, newTickUpper);
        emit RebalanceTriggered(key, reason);
    }
    
    function updateStrategyParams(
        PoolKey calldata key,
        StrategyParams calldata params
    ) external onlyOwner {
        strategyParams[key] = params;
        emit StrategyUpdated(key, params);
    }
}
```

## Развертывание и Настройка

### Последовательность Развертывания

1. **Развертывание базовых контрактов:**
   ```solidity
   // 1. StrategyManager
   StrategyManager strategyManager = new StrategyManager();
   
   // 2. UniswapV4LiquidityHook
   UniswapV4LiquidityHook hook = new UniswapV4LiquidityHook(
       IPoolManager(UNISWAP_V4_POOL_MANAGER),
       address(strategyManager)
   );
   
   // 3. ActiveLiquidityVault
   ActiveLiquidityVault vault = new ActiveLiquidityVault(
       IERC20(USDC_ADDRESS),
       "Active Liquidity Vault",
       "ALV",
       address(strategyManager),
       address(hook)
   );
   ```

2. **Настройка связей между контрактами:**
   ```solidity
   strategyManager.setVault(address(vault));
   strategyManager.setUniswapHook(address(hook));
   hook.setVault(address(vault));
   ```

3. **Настройка стратегических параметров:**
   ```solidity
   strategyManager.updateStrategyParams(poolKey, StrategyParams({
       rebalanceThreshold: 500,  // 5% приближение к границе
       apyThreshold: 1000,       // 10% минимальный APY
       maxSlippage: 100,         // 1% максимальный slippage
       autoRebalanceEnabled: true
   }));
   ```

## Безопасность

### Access Control
- **Owner-only функции:** Обновление параметров стратегий, сбор комиссий, паузы
- **Backend-only функции:** Триггеры ребалансировки, обновление позиций
- **Vault-only функции:** Создание и закрытие позиций в Uniswap

### Защиты
- **ReentrancyGuard** на всех внешних вызовах
- **Pausable** для экстренных остановок
- **SafeERC20** для всех token операций
- **Slippage protection** при ребалансировках

### Аудит Требования
- Formal verification для расчетов NAV
- Stress testing для экстремальных рыночных условий
- Code review фокус на математических вычислениях
- Integration testing с Uniswap V4 testnet

## Gas Оптимизация

### Стратегии Оптимизации
- **Batch операции** для множественных позиций
- **Packed structs** для минимизации storage slots
- **View функции** для сложных расчетов off-chain
- **Event-based** архитектура для координации с бэкендом

### Ожидаемые Gas Costs
- Депозит: ~100,000 gas
- Вывод: ~80,000 gas  
- Ребалансировка: ~200,000 gas
- Сбор комиссий: ~150,000 gas

Это обеспечивает экономическую эффективность для депозитов от $100,000.