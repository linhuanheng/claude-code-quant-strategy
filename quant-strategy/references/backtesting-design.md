# 回测框架设计

## 架构概览

```
┌──────────────────────────────────────────────────────┐
│                    BacktestEngine                      │
│                                                        │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────┐     │
│  │DataEngine│→│ Strategy │→│ PortfolioManager │     │
│  └──────────┘  └──────────┘  └────────┬─────────┘     │
│                                        ↓               │
│                               ┌────────────────┐       │
│                               │ExecutionSim    │       │
│                               └────────┬───────┘       │
│                                        ↓               │
│  ┌──────────────┐  ┌──────────────────────┐            │
│  │Performance   │← │    RiskManager       │            │
│  │Analyzer      │  │                      │            │
│  └──────────────┘  └──────────────────────┘            │
└──────────────────────────────────────────────────────┘
```

事件驱动模式回测循环：

```python
for date in trading_dates:
    # 1. 更新当前持仓市值（mark-to-market）
    portfolio.mark_to_market(date)
    
    # 2. 检查是否调仓日
    if portfolio.is_rebalance_date(date):
        # 3. 生成信号（只能使用 date 及之前的数据）
        signals = strategy.generate_signal(date)
        
        # 4. 风控检查
        signals = risk_manager.check(signals, portfolio, date)
        
        # 5. 计算目标持仓
        target_weights = portfolio_manager.compute_weights(signals, date)
        
        # 6. 生成订单
        orders = portfolio_manager.generate_orders(target_weights, date)
        
        # 7. 模拟成交
        trades = execution_sim.execute(orders, date)
        
        # 8. 更新持仓
        portfolio.update(trades, date)
    
    # 9. 记录每日净值
    portfolio.record_daily(date)
```

---

## 各模块接口定义

### DataEngine

```python
class DataEngine:
    """数据引擎：加载、存储、提供 PIT 数据切片。
    
    核心职责：确保任何时间点只能获取到当前已知的数据。
    """
    
    def __init__(self, start_date: str, end_date: str):
        self.start_date = pd.Timestamp(start_date)
        self.end_date = pd.Timestamp(end_date)
        self._data: Dict[str, pd.DataFrame] = {}
        self._trading_calendar: List[pd.Timestamp] = []
    
    def load_price(self, ts_codes: List[str]) -> None:
        """加载复权价格数据。
        
        通过 tushare MCP 的 daily + adj_factor 获取后复权价格。
        """
        ...
    
    def load_financials(self, ts_codes: List[str],
                        fields: List[str]) -> None:
        """加载财务数据，处理披露延迟。
        
        年报披露截止日为次年 4 月 30 日，
        中报为 8 月 31 日，
        季报为报告期结束后 1 个月内。
        在此日期之前使用上一期数据。
        """
        ...
    
    def get_snapshot(self, date: pd.Timestamp) -> Dict[str, pd.DataFrame]:
        """获取 date 日已知的所有数据切片（PIT 视角）。"""
        ...
    
    def get_trading_dates(self) -> List[pd.Timestamp]:
        """获取交易日历。"""
        ...
```

### Strategy

```python
class BaseStrategy(ABC):
    """策略抽象基类。"""
    
    def __init__(self, data_engine: DataEngine, config: dataclass):
        self.data = data_engine
        self.config = config
    
    @abstractmethod
    def generate_signal(self, date: pd.Timestamp) -> pd.Series:
        """在 date 日生成目标权重信号。
        
        Args:
            date: 调仓日日期。
        
        Returns:
            Series(index=ts_code, values=weight)，
            权重为正表示做多，为负表示做空。
            权重之和等于目标总敞口（如 1.0 表示满仓做多）。
            不在股票池内的股票信号为 NaN。
        """
        ...
    
    def pre_trade_check(self, date: pd.Timestamp) -> bool:
        """交易前检查：是否允许在 date 日交易。
        
        例如：市场大跌不交易、重大事件日不交易等。
        """
        return True
```

### PortfolioManager

```python
class PortfolioManager:
    """持仓管理与权重计算。"""
    
    def __init__(self, config):
        self.positions: Dict[str, float] = {}    # ts_code -> 持仓数量(股)
        self.cash: float = config.initial_capital
        self.nav_history: List[dict] = []
    
    def compute_target_weights(self, signals: pd.Series,
                                date: pd.Timestamp) -> pd.Series:
        """将原始信号转换为目标持仓权重。
        
        包含：
        - 信号截断（只取 top/bottom N）
        - 权重归一化
        - 行业/个股权重上限
        """
        ...
    
    def generate_orders(self, target_weights: pd.Series,
                         date: pd.Timestamp) -> List[Order]:
        """计算从当前持仓到目标持仓所需的订单列表。"""
        ...
    
    def update(self, trades: List[Trade], date: pd.Timestamp) -> None:
        """根据成交结果更新持仓和现金。"""
        ...
    
    def mark_to_market(self, date: pd.Timestamp) -> float:
        """按当日收盘价计算持仓市值和总净值。"""
        ...
    
    def record_daily(self, date: pd.Timestamp) -> None:
        """记录当日净值、持仓等快照数据。"""
        ...
```

### ExecutionSimulator

```python
@dataclass
class ExecutionConfig:
    commission_rate: float = 0.0003      # 双边佣金万三
    stamp_tax_rate: float = 0.0005       # 卖出印花税（仅 A 股卖出）
    slippage_bp: float = 1.0             # 滑点 1bp
    execution_price: str = "next_open"   # T+1 开盘 / T日VWAP / T日收盘
    
    # 涨跌停限制
    limit_up_check: bool = True
    limit_down_check: bool = True
    
    # 流动性限制
    max_turnover_ratio: float = 0.10     # 单日最大买卖不超过日均成交量的 10%

class ExecutionSimulator:
    """成交模拟器。
    
    模拟真实市场约束下的订单执行：
    - 滑点
    - 手续费/印花税
    - 涨跌停不可交易
    - 停牌不可交易
    - 流动性约束
    """
    
    def execute(self, orders: List[Order],
                date: pd.Timestamp) -> List[Trade]:
        """执行订单列表，返回实际成交列表。
        
        对于不能成交的订单（涨跌停/停牌/流动性不足），
        记录为未成交，不产生 Trade。
        """
        ...
    
    def get_execution_price(self, ts_code: str,
                             date: pd.Timestamp,
                             direction: str) -> float:
        """获取执行价格。
        
        根据 config.execution_price：
        - "next_open": T+1 日开盘价（推荐，可实际执行）
        - "close": T 日收盘价（有前视偏差，仅用于快速原型）
        - "vwap": T+1 日 VWAP（需要分钟数据）
        """
        ...
```

### RiskManager

```python
@dataclass
class RiskConfig:
    max_position_pct: float = 0.10        # 单只股票最大仓位
    max_industry_pct: float = 0.30        # 单行业最大仓位
    max_turnover_rate: float = 1.0        # 最大单边换手率
    stop_loss_pct: float = -0.10          # 个股止损线
    max_drawdown_pct: float = 0.20        # 组合层面最大回撤止损
    max_leverage: float = 1.0             # 最大杠杆（多空净值/权益）

class RiskManager:
    """风控模块。
    
    在信号生成后、订单执行前进行风险检查。
    可以修改目标权重、拒绝订单或强制平仓。
    """
    
    def check(self, signals: pd.Series,
              portfolio: PortfolioManager,
              date: pd.Timestamp) -> pd.Series:
        """对信号进行风控过滤。
        
        Returns:
            调整后的信号 Series。
        """
        signals = self._check_individual_limits(signals, date)
        signals = self._check_industry_exposure(signals, date)
        signals = self._check_turnover(signals, portfolio, date)
        signals = self._check_stop_loss(portfolio, date)
        return signals
    
    def check_portfolio_stop(self, portfolio: PortfolioManager,
                              date: pd.Timestamp) -> bool:
        """组合层面的止损检查。
        
        Returns:
            True 如果允许继续交易，False 如果应停止或减仓。
        """
        ...
```

### PerformanceAnalyzer

```python
class PerformanceAnalyzer:
    """绩效分析器。
    
    分析回测结果，生成完整的绩效报告。
    所有统计量必须使用 PIT 可得的收益率序列计算。
    """
    
    def __init__(self, nav_series: pd.Series,
                 benchmark_nav: pd.Series = None,
                 risk_free_rate: float = 0.02):
        self.returns = nav_series.pct_change().dropna()
        self.benchmark_returns = (benchmark_nav.pct_change().dropna()
                                  if benchmark_nav is not None else None)
        self.rf = risk_free_rate
    
    def compute_all_metrics(self) -> Dict[str, Any]:
        """计算所有绩效指标。
        
        返回完整的指标字典，涵盖：
        - 收益指标（年化收益、累计收益等）
        - 风险指标（波动率、最大回撤、VaR等）
        - 风险调整指标（Sharpe、Sortino、Calmar等）
        - 收益分布（所有分位数）
        - NW 调整后的显著性
        """
        ...
    
    def generate_report(self) -> str:
        """生成 Markdown 格式的绩效分析报告。"""
        ...
    
    def plot_nav_curve(self) -> None:
        """绘制净值曲线（对数坐标），策略 vs 基准。"""
        ...
    
    def plot_drawdown(self) -> None:
        """绘制回撤曲线。"""
        ...
    
    def plot_monthly_heatmap(self) -> None:
        """绘制月度收益热力图。"""
        ...
    
    def plot_return_distribution(self) -> None:
        """绘制收益分布直方图 + 正态拟合。"""
        ...
```

---

## 隐式假设检查清单

回测框架中隐含的假设对结果影响极大，必须明确记录：

| 假设 | 选项 | 推荐 |
|------|------|------|
| 执行价格 | T 日收盘 / T+1 开盘 / T+1 VWAP | T+1 开盘（最保守可行） |
| 手续费 | 万三 / 万一 / 自定义 | 万三双边 + 千一印花税（卖出） |
| 滑点 | 0 / 1bp / 2bp / 成交量加权 | 1-2bp |
| 涨停买入 | 允许/禁止 | 禁止 |
| 跌停卖出 | 允许/禁止 | 禁止 |
| 停牌股票 | 持有/剔除/按0收益 | 剔除，资金按比例分配 |
| 分红再投资 | 是/否 | 是（使用后复权价格） |
| 最小交易单位 | 100 股 | 100 股（A 股） |
| 资金不足时的处理 | 按比例缩/跳过该股票 | 按比例缩 |
| 退市股票 | 最后一天按收盘价卖出 / 亏损全部 | 最后一天按收盘价卖出 |

---

## 常见错误警示

1. **使用 T 日收盘价作为执行价格**：实际中 T 日收盘产生信号，只能以 T+1 日价格成交
2. **忽略涨跌停**：涨停时无法买入，跌停时无法卖出
3. **不处理除权除息**：导致虚假收益（用后复权价格避免）
4. **全样本标准化**：因子标准化用了未来的均值/标准差
5. **幸存者偏差**：回测股票池只包含存活至今的股票（应包含已退市股票）
6. **前视偏差**：T 日使用了 T 日收盘后才公布的数据
