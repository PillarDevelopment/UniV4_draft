# Frontend Спецификация: Uniswap V4 Активное Управление Ликвидностью

## Обзор

Frontend система состоит из двух основных интерфейсов:
1. **Пользовательский интерфейс** — простой dashboard для инвесторов (баланс + APY)
2. **Панель стратегиста** — профессиональный инструмент для управления позициями и стратегиями

## Архитектура Frontend

### Техническая Архитектура

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Next.js App   │    │  State Management│    │   Web3 Layer    │
│                 │◄──►│                 │◄──►│                 │
│ - SSR/SSG       │    │ - Zustand       │    │ - Wagmi         │
│ - API Routes    │    │ - React Query   │    │ - Viem          │
│ - Middleware    │    │ - Persistence   │    │ - WalletConnect │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         ▼                       ▼                       ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   UI Components │    │   Data Layer    │    │   Security      │
│                 │    │                 │    │                 │
│ - Shadcn/ui     │    │ - API Client    │    │ - Auth Guard    │
│ - Tailwind      │    │ - WebSocket     │    │ - Role-based    │
│ - Recharts      │    │ - Real-time     │    │ - Rate Limiting │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

## Пользовательский Интерфейс

### 1. Главная Страница (Dashboard)

**Макет:**
```tsx
// pages/dashboard.tsx
import React from 'react';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Button } from '@/components/ui/button';
import { useVaultData } from '@/hooks/useVaultData';
import { useUserPortfolio } from '@/hooks/useUserPortfolio';

export default function Dashboard() {
  const { vaultData, isLoading: vaultLoading } = useVaultData();
  const { portfolio, isLoading: portfolioLoading } = useUserPortfolio();
  
  if (vaultLoading || portfolioLoading) {
    return <DashboardSkeleton />;
  }
  
  return (
    <div className="container mx-auto p-6 max-w-4xl">
      <div className="mb-8">
        <h1 className="text-3xl font-bold text-gray-900 dark:text-gray-100">
          Liquidity Vault Dashboard
        </h1>
        <p className="text-gray-600 dark:text-gray-400 mt-2">
          Professional Uniswap V4 liquidity management
        </p>
      </div>
      
      {/* Баланс и основные метрики */}
      <div className="grid grid-cols-1 md:grid-cols-2 gap-6 mb-8">
        <BalanceCard portfolio={portfolio} />
        <APYCard vaultData={vaultData} portfolio={portfolio} />
      </div>
      
      {/* График производительности */}
      <div className="mb-8">
        <PerformanceChart portfolio={portfolio} />
      </div>
      
      {/* Действия пользователя */}
      <div className="grid grid-cols-1 md:grid-cols-2 gap-6">
        <DepositCard vaultData={vaultData} />
        <WithdrawCard portfolio={portfolio} vaultData={vaultData} />
      </div>
    </div>
  );
}
```

### 2. Компонент Баланса

```tsx
// components/BalanceCard.tsx
import React from 'react';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { DollarSign, TrendingUp } from 'lucide-react';
import { formatCurrency, formatPercentage } from '@/utils/formatters';

interface BalanceCardProps {
  portfolio: UserPortfolio;
}

export function BalanceCard({ portfolio }: BalanceCardProps) {
  return (
    <Card>
      <CardHeader className="flex flex-row items-center justify-between space-y-0 pb-2">
        <CardTitle className="text-sm font-medium">Your Balance</CardTitle>
        <DollarSign className="h-4 w-4 text-muted-foreground" />
      </CardHeader>
      <CardContent>
        <div className="text-2xl font-bold">
          {formatCurrency(portfolio.balance.valueUSD)}
        </div>
        <div className="flex items-center text-xs text-muted-foreground mt-1">
          <span>
            {formatCurrency(portfolio.balance.shares, 4)} shares 
            ({formatPercentage(portfolio.balance.percentage)}% of vault)
          </span>
        </div>
        
        {/* Изменение за последние 24ч */}
        <div className="flex items-center mt-3">
          <TrendingUp className="h-3 w-3 text-green-500 mr-1" />
          <span className="text-xs text-green-600">
            +{formatPercentage(portfolio.performance.dailyChange)}% (24h)
          </span>
        </div>
      </CardContent>
    </Card>
  );
}
```

### 3. Компонент APY

```tsx
// components/APY
. Компонент APY
tsx// components/APYCard.tsx
import React from 'react';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Percent, Info } from 'lucide-react';
import { formatPercentage } from '@/utils/formatters';
import { Tooltip, TooltipContent, TooltipProvider, TooltipTrigger } from '@/components/ui/tooltip';

interface APYCardProps {
  vaultData: VaultData;
  portfolio: UserPortfolio;
}

export function APYCard({ vaultData, portfolio }: APYCardProps) {
  return (
    <Card>
      <CardHeader className="flex flex-row items-center justify-between space-y-0 pb-2">
        <CardTitle className="text-sm font-medium flex items-center">
          Annual Percentage Yield
          <TooltipProvider>
            <Tooltip>
              <TooltipTrigger>
                <Info className="h-3 w-3 text-muted-foreground ml-1" />
              </TooltipTrigger>
              <TooltipContent>
                <p>APY включает fees от Uniswap и результаты хеджирования</p>
              </TooltipContent>
            </Tooltip>
          </TooltipProvider>
        </CardTitle>
        <Percent className="h-4 w-4 text-muted-foreground" />
      </CardHeader>
      <CardContent>
        <div className="text-2xl font-bold text-green-600">
          {formatPercentage(portfolio.performance.apy)}%
        </div>
        
        {/* APY периоды */}
        <div className="grid grid-cols-3 gap-2 mt-4 text-xs text-muted-foreground">
          <div>
            <div className="font-medium">7D</div>
            <div>{formatPercentage(vaultData.performance.apy7d)}%</div>
          </div>
          <div>
            <div className="font-medium">30D</div>
            <div>{formatPercentage(vaultData.performance.apy30d)}%</div>
          </div>
          <div>
            <div className="font-medium">Since Entry</div>
            <div>{formatPercentage(portfolio.performance.totalReturn)}%</div>
          </div>
        </div>
        
        {/* Сравнение с базовой LP стратегией */}
        <div className="mt-3 p-2 bg-green-50 rounded-md">
          <div className="text-xs text-green-700">
            vs Base LP: +{formatPercentage(vaultData.performance.vsBasicLP)}%
          </div>
        </div>
      </CardContent>
    </Card>
  );
}
4. График Производительности
tsx// components/PerformanceChart.tsx
import React, { useState } from 'react';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { LineChart, Line, XAxis, YAxis, CartesianGrid, Tooltip, ResponsiveContainer } from 'recharts';
import { Button } from '@/components/ui/button';
import { usePerformanceHistory } from '@/hooks/usePerformanceHistory';

interface PerformanceChartProps {
  portfolio: UserPortfolio;
}

export function PerformanceChart({ portfolio }: PerformanceChartProps) {
  const [timeframe, setTimeframe] = useState<'7d' | '30d' | '90d'>('30d');
  const { data: chartData, isLoading } = usePerformanceHistory(portfolio.userId, timeframe);
  
  return (
    <Card>
      <CardHeader>
        <div className="flex justify-between items-center">
          <CardTitle>Portfolio Performance</CardTitle>
          <div className="flex space-x-2">
            {(['7d', '30d', '90d'] as const).map((period) => (
              <Button
                key={period}
                variant={timeframe === period ? 'default' : 'outline'}
                size="sm"
                onClick={() => setTimeframe(period)}
              >
                {period.toUpperCase()}
              </Button>
            ))}
          </div>
        </div>
      </CardHeader>
      <CardContent>
        <div className="h-64 w-full">
          {isLoading ? (
            <div className="flex items-center justify-center h-full">
              <div className="animate-spin rounded-full h-8 w-8 border-b-2 border-blue-500" />
            </div>
          ) : (
            <ResponsiveContainer width="100%" height="100%">
              <LineChart data={chartData}>
                <CartesianGrid strokeDasharray="3 3" className="stroke-gray-200" />
                <XAxis 
                  dataKey="date" 
                  tick={{ fontSize: 12 }}
                  tickFormatter={(date) => new Date(date).toLocaleDateString('en-US', { month: 'short', day: 'numeric' })}
                />
                <YAxis 
                  tick={{ fontSize: 12 }}
                  tickFormatter={(value) => `${(value / 1000).toFixed(0)}k`}
                />
                <Tooltip
                  formatter={(value: number) => [`${value.toLocaleString()}`, 'Portfolio Value']}
                  labelFormatter={(date) => new Date(date).toLocaleDateString()}
                />
                <Line 
                  type="monotone" 
                  dataKey="portfolioValue" 
                  stroke="#3b82f6" 
                  strokeWidth={2}
                  dot={false}
                />
              </LineChart>
            </ResponsiveContainer>
          )}
        </div>
      </CardContent>
    </Card>
  );
}
5. Компонент Депозита
tsx// components/DepositCard.tsx
import React, { useState } from 'react';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { ArrowDownCircle, AlertCircle } from 'lucide-react';
import { useDeposit } from '@/hooks/useDeposit';
import { formatCurrency } from '@/utils/formatters';
import { Alert, AlertDescription } from '@/components/ui/alert';

interface DepositCardProps {
  vaultData: VaultData;
}

export function DepositCard({ vaultData }: DepositCardProps) {
  const [amount, setAmount] = useState('');
  const { deposit, isLoading, error } = useDeposit();
  const [isValidAmount, setIsValidAmount] = useState(false);
  
  const minDeposit = 100000; // $100,000
  const amountNumber = parseFloat(amount) || 0;
  
  React.useEffect(() => {
    setIsValidAmount(amountNumber >= minDeposit);
  }, [amountNumber]);
  
  const handleDeposit = async () => {
    if (!isValidAmount) return;
    
    try {
      await deposit(amountNumber);
      setAmount('');
    } catch (err) {
      console.error('Deposit failed:', err);
    }
  };
  
  return (
    <Card>
      <CardHeader>
        <CardTitle className="flex items-center">
          <ArrowDownCircle className="h-5 w-5 mr-2 text-green-600" />
          Deposit
        </CardTitle>
      </CardHeader>
      <CardContent className="space-y-4">
        {/* Минимальная сумма */}
        <Alert>
          <AlertCircle className="h-4 w-4" />
          <AlertDescription>
            Minimum deposit: {formatCurrency(minDeposit)}
          </AlertDescription>
        </Alert>
        
        {/* Ввод суммы */}
        <div className="space-y-2">
          <label className="text-sm font-medium">Amount (USDC)</label>
          <div className="relative">
            <Input
              type="number"
              placeholder="0.00"
              value={amount}
              onChange={(e) => setAmount(e.target.value)}
              className="pl-8"
            />
            <span className="absolute left-3 top-1/2 transform -translate-y-1/2 text-gray-400">$</span>
          </div>
          {amount && !isValidAmount && (
            <p className="text-sm text-red-500">
              Amount must be at least {formatCurrency(minDeposit)}
            </p>
          )}
        </div>
        
        {/* Прогноз получаемых shares */}
        {isValidAmount && (
          <div className="p-3 bg-gray-50 rounded-md">
            <div className="text-sm text-gray-600">You will receive approximately:</div>
            <div className="font-semibold">
              {formatCurrency(amountNumber / vaultData.sharePrice, 4)} vault shares
            </div>
          </div>
        )}
        
        {/* Кнопка депозита */}
        <Button 
          className="w-full" 
          onClick={handleDeposit}
          disabled={!isValidAmount || isLoading}
        >
          {isLoading ? 'Depositing...' : 'Deposit'}
        </Button>
        
        {error && (
          <Alert variant="destructive">
            <AlertDescription>{error}</AlertDescription>
          </Alert>
        )}
      </CardContent>
    </Card>
  );
}
6. Компонент Вывода
tsx// components/WithdrawCard.tsx
import React, { useState } from 'react';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { ArrowUpCircle, Clock, AlertTriangle } from 'lucide-react';
import { useWithdraw } from '@/hooks/useWithdraw';
import { formatCurrency, formatDate } from '@/utils/formatters';
import { Alert, AlertDescription } from '@/components/ui/alert';

interface WithdrawCardProps {
  portfolio: UserPortfolio;
  vaultData: VaultData;
}

export function WithdrawCard({ portfolio, vaultData }: WithdrawCardProps) {
  const [shareAmount, setShareAmount] = useState('');
  const { withdraw, isLoading, error } = useWithdraw();
  
  const shareAmountNumber = parseFloat(shareAmount) || 0;
  const maxShares = portfolio.balance.shares;
  const isValidAmount = shareAmountNumber > 0 && shareAmountNumber <= maxShares;
  const estimatedUSD = shareAmountNumber * vaultData.sharePrice;
  
  const handleWithdraw = async () => {
    if (!isValidAmount) return;
    
    try {
      await withdraw(shareAmountNumber);
      setShareAmount('');
    } catch (err) {
      console.error('Withdrawal failed:', err);
    }
  };
  
  const isWithdrawalWindowOpen = vaultData.withdrawalWindow?.isOpen;
  const nextWindowDate = vaultData.withdrawalWindow?.nextOpenDate;
  
  return (
    <Card>
      <CardHeader>
        <CardTitle className="flex items-center">
          <ArrowUpCircle className="h-5 w-5 mr-2 text-blue-600" />
          Withdraw
        </CardTitle>
      </CardHeader>
      <CardContent className="space-y-4">
        {/* Статус окна вывода */}
        {!isWithdrawalWindowOpen && (
          <Alert>
            <Clock className="h-4 w-4" />
            <AlertDescription>
              Next withdrawal window: {formatDate(nextWindowDate)}
            </AlertDescription>
          </Alert>
        )}
        
        {isWithdrawalWindowOpen && (
          <Alert>
            <AlertTriangle className="h-4 w-4" />
            <AlertDescription>
              Withdrawal window open until {formatDate(vaultData.withdrawalWindow.closeDate)}
            </AlertDescription>
          </Alert>
        )}
        
        {/* Ввод количества shares */}
        <div className="space-y-2">
          <label className="text-sm font-medium">Shares to withdraw</label>
          <div className="relative">
            <Input
              type="number"
              placeholder="0.0000"
              value={shareAmount}
              onChange={(e) => setShareAmount(e.target.value)}
              max={maxShares}
              step="0.0001"
              disabled={!isWithdrawalWindowOpen}
            />
            <Button
              variant="ghost"
              size="sm"
              className="absolute right-2 top-1/2 transform -translate-y-1/2 h-6 px-2 text-xs"
              onClick={() => setShareAmount(maxShares.toString())}
              disabled={!isWithdrawalWindowOpen}
            >
              MAX
            </Button>
          </div>
          <div className="flex justify-between text-xs text-gray-500">
            <span>Available: {formatCurrency(maxShares, 4)} shares</span>
            <span>{((shareAmountNumber / maxShares) * 100 || 0).toFixed(1)}%</span>
          </div>
        </div>
        
        {/* Прогноз получаемой суммы */}
        {isValidAmount && (
          <div className="p-3 bg-gray-50 rounded-md">
            <div className="text-sm text-gray-600">You will receive approximately:</div>
            <div className="font-semibold">{formatCurrency(estimatedUSD)}</div>
          </div>
        )}
        
        {/* Кнопка вывода */}
        <Button 
          className="w-full" 
          onClick={handleWithdraw}
          disabled={!isValidAmount || !isWithdrawalWindowOpen || isLoading}
        >
          {isLoading ? 'Processing...' : 'Withdraw'}
        </Button>
        
        {error && (
          <Alert variant="destructive">
            <AlertDescription>{error}</AlertDescription>
          </Alert>
        )}
      </CardContent>
    </Card>
  );
}
Панель Стратегиста
1. Главная Панель Управления
tsx// pages/strategy/dashboard.tsx
import React from 'react';
import { useStrategyData } from '@/hooks/useStrategyData';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Tabs, TabsContent, TabsList, TabsTrigger } from '@/components/ui/tabs';

export default function StrategyDashboard() {
  const { positions, performance, risks } = useStrategyData();
  
  return (
    <div className="container mx-auto p-6">
      <div className="mb-8">
        <h1 className="text-3xl font-bold">Strategy Management</h1>
        <p className="text-gray-600 mt-2">Active liquidity position management</p>
      </div>
      
      <Tabs defaultValue="positions" className="space-y-6">
        <TabsList className="grid w-full grid-cols-4">
          <TabsTrigger value="positions">Positions</TabsTrigger>
          <TabsTrigger value="risk">Risk Monitor</TabsTrigger>
          <TabsTrigger value="pnl">P&L Analysis</TabsTrigger>
          <TabsTrigger value="settings">Settings</TabsTrigger>
        </TabsList>
        
        <TabsContent value="positions" className="space-y-6">
          <PositionsOverview positions={positions} />
          <ActivePositionsTable positions={positions} />
        </TabsContent>
        
        <TabsContent value="risk" className="space-y-6">
          <RiskMetrics risks={risks} />
          <ILMonitoring positions={positions} />
        </TabsContent>
        
        <TabsContent value="pnl" className="space-y-6">
          <PnLSummary performance={performance} />
          <StrategyComparison />
        </TabsContent>
        
        <TabsContent value="settings" className="space-y-6">
          <StrategyParameters />
          <RebalanceSettings />
        </TabsContent>
      </Tabs>
    </div>
  );
}
2. Обзор Позиций
tsx// components/strategy/PositionsOverview.tsx
import React from 'react';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Badge } from '@/components/ui/badge';
import { Activity, TrendingUp, DollarSign, Zap } from 'lucide-react';

interface PositionsOverviewProps {
  positions: StrategyPosition[];
}

export function PositionsOverview({ positions }: PositionsOverviewProps) {
  const activePositions = positions.filter(p => p.status === 'active');
  const totalLiquidity = positions.reduce((sum, p) => sum + p.liquidityUSD, 0);
  const avgAPY = positions.reduce((sum, p) => sum + p.currentAPY, 0) / positions.length;
  const totalIL = positions.reduce((sum, p) => sum + p.impermanentLoss, 0);
  
  return (
    <div className="grid grid-cols-1 md:grid-cols-4 gap-6">
      <Card>
        <CardHeader className="flex flex-row items-center justify-between space-y-0 pb-2">
          <CardTitle className="text-sm font-medium">Active Positions</CardTitle>
          <Activity className="h-4 w-4 text-muted-foreground" />
        </CardHeader>
        <CardContent>
          <div className="text-2xl font-bold">{activePositions.length}</div>
          <p className="text-xs text-muted-foreground">
            {positions.length - activePositions.length} rebalancing
          </p>
        </CardContent>
      </Card>
      
      <Card>
        <CardHeader className="flex flex-row items-center justify-between space-y-0 pb-2">
          <CardTitle className="text-sm font-medium">Total Liquidity</CardTitle>
          <DollarSign className="h-4 w-4 text-muted-foreground" />
        </CardHeader>
        <CardContent>
          <div className="text-2xl font-bold">
            {formatCurrency(totalLiquidity)}
          </div>
          <p className="text-xs text-muted-foreground">
            Across {activePositions.length} pools
          </p>
        </CardContent>
      </Card>
      
      <Card>
        <CardHeader className="flex flex-row items-center justify-between space-y-0 pb-2">
          <CardTitle className="text-sm font-medium">Average APY</CardTitle>
          <TrendingUp className="h-4 w-4 text-muted-foreground" />
        </CardHeader>
        <CardContent>
          <div className="text-2xl font-bold text-green-600">
            {formatPercentage(avgAPY)}%
          </div>
          <p className="text-xs text-muted-foreground">
            Last 30 days
          </p>
        </CardContent>
      </Card>
      
      <Card>
        <CardHeader className="flex flex-row items-center justify-between space-y-0 pb-2">
          <CardTitle className="text-sm font-medium">Total IL</CardTitle>
          <Zap className="h-4 w-4 text-muted-foreground" />
        </CardHeader>
        <CardContent>
          <div className="text-2xl font-bold text-red-600">
            {formatCurrency(totalIL)}
          </div>
          <p className="text-xs text-muted-foreground">
            {formatPercentage((totalIL / totalLiquidity) * 100)}% of liquidity
          </p>
        </CardContent>
      </Card>
    </div>
  );
}
3. Таблица Активных Позиций
tsx// components/strategy/ActivePositionsTable.tsx
import React from 'react';
import {
  Table,
  TableBody,
  TableCell,
  TableHead,
  TableHeader,
  TableRow,
} from '@/components/ui/table';
import { Button } from '@/components/ui/button';
import { Badge } from '@/components/ui/badge';
import { MoreHorizontal, TrendingUp, TrendingDown } from 'lucide-react';
import {
  DropdownMenu,
  DropdownMenuContent,
  DropdownMenuItem,
  DropdownMenuTrigger,
} from '@/components/ui/dropdown-menu';

interface ActivePositionsTableProps {
  positions: StrategyPosition[];
}

export function ActivePositionsTable({ positions }: ActivePositionsTableProps) {
  const [sortBy, setSortBy] = useState<'apy' | 'liquidity' | 'il'>('apy');
  
  const sortedPositions = [...positions].sort((a, b) => {
    switch (sortBy) {
      case 'apy':
        return b.currentAPY - a.currentAPY;
      case 'liquidity':
        return b.liquidityUSD - a.liquidityUSD;
      case 'il':
        return b.impermanentLoss - a.impermanentLoss;
      default:
        return 0;
    }
  });
  
  const getStatusBadge = (status: string, health: number) => {
    if (status === 'rebalancing') {
      return <Badge variant="outline" className="bg-yellow-50">Rebalancing</Badge>;
    }
    
    if (health > 0.8) {
      return <Badge variant="outline" className="bg-green-50 text-green-700">Healthy</Badge>;
    } else if (health > 0.5) {
      return <Badge variant="outline" className="bg-yellow-50 text-yellow-700">Warning</Badge>;
    } else {
      return <Badge variant="outline" className="bg-red-50 text-red-700">Risk</Badge>;
    }
  };
  
  return (
    <Card>
      <CardHeader>
        <div className="flex justify-between items-center">
          <CardTitle>Active Positions</CardTitle>
          <div className="flex space-x-2">
            <Button variant="outline" size="sm" onClick={() => setSortBy('apy')}>
              Sort by APY
            </Button>
            <Button variant="outline" size="sm" onClick={() => setSortBy('liquidity')}>
              Sort by Size
            </Button>
          </div>
        </div>
      </CardHeader>
      <CardContent>
        <Table>
          <TableHeader>
            <TableRow>
              <TableHead>Pool</TableHead>
              <TableHead>Range</TableHead>
              <TableHead>Liquidity</TableHead>
              <TableHead>Current APY</TableHead>
              <TableHead>IL</TableHead>
              <TableHead>Status</TableHead>
              <TableHead>Actions</TableHead>
            </TableRow>
          </TableHeader>
          <TableBody>
            {sortedPositions.map((position) => (
              <TableRow key={position.id}>
                <TableCell>
                  <div className="flex items-center space-x-2">
                    <div className="flex -space-x-1">
                      <img
                        src={position.pool.token0.logo}
                        alt={position.pool.token0.symbol}
                        className="w-6 h-6 rounded-full border-2 border-white"
                      />
                      <img
                        src={position.pool.token1.logo}
                        alt={position.pool.token1.symbol}
                        className="w-6 h-6 rounded-full border-2 border-white"
                      />
                    </div>
                    <span className="font-medium">
                      {position.pool.token0.symbol}/{position.pool.token1.symbol}
                    </span>
                  </div>
                </TableCell>
                
                <TableCell>
                  <div className="text-sm">
                    <div>{position.priceRange.lower.toFixed(4)}</div>
                    <div className="text-gray-500">
                      {position.priceRange.upper.toFixed(4)}
                    </div>
                  </div>
                </TableCell>
                
                <TableCell>
                  <div className="text-sm">
                    <div className="font-medium">
                      {formatCurrency(position.liquidityUSD)}
                    </div>
                    <div className="text-gray-500">
                      {formatPercentage((position.liquidityUSD / totalLiquidity) * 100)}%
                    </div>
                  </div>
                </TableCell>
                
                <TableCell>
                  <div className="flex items-center">
                    <span className="font-medium text-green-600">
                      {formatPercentage(position.currentAPY)}%
                    </span>
                    {position.apyTrend === 'up' ? (
                      <TrendingUp className="h-3 w-3 text-green-500 ml-1" />
                    ) : (
                      <TrendingDown className="h-3 w-3 text-red-500 ml-1" />
                    )}
                  </div>
                </TableCell>
                
                <TableCell>
                  <div className="text-sm">
                    <div className={position.impermanentLoss > 0 ? 'text-red-600' : 'text-green-600'}>
                      {formatCurrency(position.impermanentLoss)}
                    </div>
                    <div className="text-gray-500">
                      {formatPercentage((position.impermanentLoss / position.liquidityUSD) * 100)}%
                    </div>
                  </div>
                </TableCell>
                
                <TableCell>
                  {getStatusBadge(position.status, position.healthScore)}
                </TableCell>
                
                <TableCell>
                  <DropdownMenu>
                    <DropdownMenuTrigger asChild>
                      <Button variant="ghost" className="h-8 w-8 p-0">
                        <MoreHorizontal className="h-4 w-4" />
                      </Button>
                    </DropdownMenuTrigger>
                    <DropdownMenuContent align="end">
                      <DropdownMenuItem onClick={() => handleRebalance(position.id)}>
                        Force Rebalance
                      </DropdownMenuItem>
                      <DropdownMenuItem onClick={() => handleClosePosition(position.id)}>
                        Close Position
                      </DropdownMenuItem>
                      <DropdownMenuItem onClick={() => handleViewDetails(position.id)}>
                        View Details
                      </DropdownMenuItem>
                    </DropdownMenuContent>
                  </DropdownMenu>
                </TableCell>
              </TableRow>
            ))}
          </TableBody>
        </Table>
      </CardContent>
    </Card>
  );
}
4. Мониторинг Рисков
tsx// components/strategy/RiskMetrics.tsx
import React from 'react';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Progress } from '@/components/ui/progress';
import { Alert, AlertDescription } from '@/components/ui/alert';
import { AlertTriangle, Shield, TrendingDown } from 'lucide-react';

interface RiskMetricsProps {
  risks: RiskData;
}

export function RiskMetrics({ risks }: RiskMetricsProps) {
  const getRiskColor = (value: number, threshold: number) => {
    if (value < threshold * 0.5) return 'text-green-600';
    if (value < threshold * 0.8) return 'text-yellow-600';
    return 'text-red-600';
  };
  
  const getRiskBg = (value: number, threshold: number) => {
    if (value < threshold * 0.5) return 'bg-green-100';
    if (value < threshold * 0.8) return 'bg-yellow-100';
    return 'bg-red-100';
  };
  
  return (
    <div className="grid grid-cols-1 md:grid-cols-3 gap-6">
      {/* Максимальная просадка */}
      <Card>
        <CardHeader>
          <CardTitle className="flex items-center text-sm">
            <TrendingDown className="h-4 w-4 mr-2" />
            Max Drawdown
          </CardTitle>
        </CardHeader>
        <CardContent>
          <div className="space-y-3">
            <div className="flex justify-between items-center">
              <span className="text-2xl font-bold">
                {formatPercentage(risks.maxDrawdown)}%
              </span>
              <span className={`text-sm ${getRiskColor(risks.maxDrawdown, 0.15)}`}>
                {risks.maxDrawdown < 0.10 ? 'Low Risk' : 
                 risks.maxDrawdown < 0.15 ? 'Medium Risk' : 'High Risk'}
              </span>
            </div>
            <Progress 
              value={(risks.maxDrawdown / 0.25) * 100} 
              className={`h-2 ${getRiskBg(risks.maxDrawdown, 0.15)}`}
            />
            <p className="text-xs text-gray-500">
              Limit: 15% | Current: {formatPercentage(risks.maxDrawdown)}%
            </p>
          </div>
        </CardContent>
      </Card>
      
      {/* VaR (Value at Risk) */}
      <Card>
        <CardHeader>
          <CardTitle className="flex items-center text-sm">
            <AlertTriangle className="h-4 w-4 mr-2" />
            Value at Risk (95%)
          </CardTitle>
        </CardHeader>
        <CardContent>
          <div className="space-y-3">
            <div className="flex justify-between items-center">
              <span className="text-2xl font-bold">
                {formatCurrency(risks.valueAtRisk)}
              </span>
              <span className={`text-sm ${getRiskColor(risks.valueAtRisk, 500000)}`}>
                1-Day
              </span>
            </div>
            <Progress 
              value={(risks.valueAtRisk / 1000000) * 100} 
              className="h-2"
            />
            <p className="text-xs text-gray-500">
              95% confidence | 1-day horizon
            </p>
          </div># Frontend Спецификация: Uniswap V4 Активное Управление Ликвидностью

## Обзор

Frontend система состоит из двух основных интерфейсов:
1. **Пользовательский интерфейс** — простой dashboard для инвесторов (баланс + APY)
2. **Панель стратегиста** — профессиональный инструмент для управления позициями и стратегиями

## Архитектура Frontend

### Техническая Архитектура
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Next.js App   │    │  State Management│    │   Web3 Layer    │
│                 │◄──►│                 │◄──►│                 │
│ - SSR/SSG       │    │ - Zustand       │    │ - Wagmi         │
│ - API Routes    │    │ - React Query   │    │ - Viem          │
│ - Middleware    │    │ - Persistence   │    │ - WalletConnect │
└─────────────────┘    └─────────────────┘    └─────────────────┘
│                       │                       │
▼                       ▼                       ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   UI Components │    │   Data Layer    │    │   Security      │
│                 │    │                 │    │                 │
│ - Shadcn/ui     │    │ - API Client    │    │ - Auth Guard    │
│ - Tailwind      │    │ - WebSocket     │    │ - Role-based    │
│ - Recharts      │    │ - Real-time     │    │ - Rate Limiting │
└─────────────────┘    └─────────────────┘    └─────────────────┘

## Пользовательский Интерфейс

### 1. Главная Страница (Dashboard)

**Макет:**
```tsx
// pages/dashboard.tsx
import React from 'react';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Button } from '@/components/ui/button';
import { useVaultData } from '@/hooks/useVaultData';
import { useUserPortfolio } from '@/hooks/useUserPortfolio';

export default function Dashboard() {
  const { vaultData, isLoading: vaultLoading } = useVaultData();
  const { portfolio, isLoading: portfolioLoading } = useUserPortfolio();
  
  if (vaultLoading || portfolioLoading) {
    return <DashboardSkeleton />;
  }
  
  return (
    <div className="container mx-auto p-6 max-w-4xl">
      <div className="mb-8">
        <h1 className="text-3xl font-bold text-gray-900 dark:text-gray-100">
          Liquidity Vault Dashboard
        </h1>
        <p className="text-gray-600 dark:text-gray-400 mt-2">
          Professional Uniswap V4 liquidity management
        </p>
      </div>
      
      {/* Баланс и основные метрики */}
      <div className="grid grid-cols-1 md:grid-cols-2 gap-6 mb-8">
        <BalanceCard portfolio={portfolio} />
        <APYCard vaultData={vaultData} portfolio={portfolio} />
      </div>
      
      {/* График производительности */}
      <div className="mb-8">
        <PerformanceChart portfolio={portfolio} />
      </div>
      
      {/* Действия пользователя */}
      <div className="grid grid-cols-1 md:grid-cols-2 gap-6">
        <DepositCard vaultData={vaultData} />
        <WithdrawCard portfolio={portfolio} vaultData={vaultData} />
      </div>
    </div>
  );
}
2. Компонент Баланса
tsx// components/BalanceCard.tsx
import React from 'react';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { DollarSign, TrendingUp } from 'lucide-react';
import { formatCurrency, formatPercentage } from '@/utils/formatters';

interface BalanceCardProps {
  portfolio: UserPortfolio;
}

export function BalanceCard({ portfolio }: BalanceCardProps) {
  return (
    <Card>
      <CardHeader className="flex flex-row items-center justify-between space-y-0 pb-2">
        <CardTitle className="text-sm font-medium">Your Balance</CardTitle>
        <DollarSign className="h-4 w-4 text-muted-foreground" />
      </CardHeader>
      <CardContent>
        <div className="text-2xl font-bold">
          {formatCurrency(portfolio.balance.valueUSD)}
        </div>
        <div className="flex items-center text-xs text-muted-foreground mt-1">
          <span>
            {formatCurrency(portfolio.balance.shares, 4)} shares 
            ({formatPercentage(portfolio.balance.percentage)}% of vault)
          </span>
        </div>
        
        {/* Изменение за последние 24ч */}
        <div className="flex items-center mt-3">
          <TrendingUp className="h-3 w-3 text-green-500 mr-1" />
          <span className="text-xs text-green-600">
            +{formatPercentage(portfolio.performance.dailyChange)}% (24h)
          </span>
        </div>
      </CardContent>
    </Card>
  );
}