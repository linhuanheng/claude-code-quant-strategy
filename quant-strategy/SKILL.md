---
name: quant-strategy
description: 为量化交易策略项目生成完整的初始代码框架。当用户提及新建量化策略、策略项目初始化、搭建回测框架、量化项目脚手架、开始写一个策略等意图时触发。覆盖选股/择时/套利/事件驱动/CTA/ML等策略类型，自动生成数据引擎、因子模块、回测引擎、绩效分析等模块的 skeleton 代码、配置文件和项目文档。
---

# 量化策略项目初始化 Skill

## 职责定位

本 skill 的职责是**帮助用户完成一个量化交易策略项目的初始化**——生成完整的项目骨架代码、配置和文档。在此之后，用户即可在这个框架基础上填充具体的策略逻辑。

不负责：实际数据拉取、因子调优、实盘部署。这些是用户在框架内自己要做的事。

## 开发日志 Agent（全程记录）

本 skill 包含一个开发日志记录子 agent（`references/dev-logger.md`），忠实地记录整个开发过程的每一次改动。

### 调用时机

以下操作完成后**必须调用日志 Agent** 记录：

- 创建、修改或删除任何 `.py` 文件
- 修改 `config.py` 中的参数（记录旧值 → 新值）
- 运行回测（记录起止日期和主要绩效数字）
- 审计 Agent 完成检查（记录审计结论）
- 修复前视偏差（记录修复了哪些违规）
- 用户提出新策略想法并落实

### 操作方式

使用 Agent 工具，`subagent_type="general-purpose"`，prompt 中指定读取 `references/dev-logger.md`，传入本次改动的描述。日志 Agent 会将记录追加到项目根目录的 `DEV_LOG.md` 文件中，精确到分钟。

### 日志格式

每条记录包含：序号、时间戳（精确到分钟）、改动类型、涉及文件、改动摘要、改动详情、结果、备注。日志追加写入，不修改已有记录。

## Look-ahead 审计 Agent（前置关卡）

本 skill 包含一个专用的审计子 agent（`references/lookahead-auditor.md`），专门检查代码是否引入了未来信息。

**审计 Agent 是收益分析的前置关卡——不通过审计，不出分析结果。**

### 触发时机

以下任一情况发生时，**必须先启动审计 Agent，审计通过后才能向用户输出结果**：

| 触发场景 | 审计范围 | 优先级 |
|---------|---------|--------|
| 用户要求查看收益分析/回测结果/绩效表现 | 涉及收益计算的全部 `.py` 文件 | 强制 |
| 用户要求核对策略表现 | 策略模块 + 回测模块 + 分析模块 | 强制 |
| 项目初始化生成全部代码后 | 所有生成的文件 | 强制 |
| 用户修改了涉及数据或计算的代码后 | 变更文件 + 受影响的调用链文件 | 强制 |
| 用户主动要求审计 | 用户指定的范围 | 按需 |

**关键约束**：在审计通过之前，不得向用户展示任何收益数字、分布统计、显著性结果、净值曲线。如果审计发现问题，必须先向用户说明问题、修复、再审计通过后，才展示分析结果。

### 审计流程

```
用户请求收益分析
        │
        ▼
  主 Agent 准备分析代码和计算
        │
        ▼
  启动审计 Agent ──────────────────────┐
  （读取 lookahead-auditor.md）         │
        │                               │
    ┌───┴───┐                           │
    │ 发现   │                           │
    │ 违规？ │                           │
    └───┬───┘                           │
        │                               │
   有违规 │ 无违规                        │
        ▼                               ▼
  主 Agent 向用户说明        ┌──────────────────┐
  违规 + 修复               │ 审计通过          │
        │                   │ 向用户展示分析结果 │
        ▼                   └──────────────────┘
  再次审计（复检）
        │
    ┌───┴───┐
    │ 仍有   │──→ 继续修复
    │ 违规？ │
    └───┬───┘
        │
   无违规──→ 向用户展示分析结果
```

### 调用方式

使用 Agent 工具，`subagent_type="general-purpose"`，prompt 中指定读取 `references/lookahead-auditor.md`，传入待审计的文件路径列表。审计 Agent 只读代码、报告违规，不修改代码。

### 审计后的处理

1. **审计通过（违规数 = 0）**：向用户报告「审计通过，未发现前视偏差」，然后展示分析结果
2. **审计未通过**：逐条向用户说明每处违规的类型、位置、影响和修复方案，修复后自动再次审计。不得跳过修复直接展示结果
3. **多次修复仍未通过**：请用户介入决策

## 核心设计原则

生成的框架代码必须内建以下原则：

### 禁止未来信息

所有模块的接口和默认实现必须遵循 point-in-time 约束：任意时间点的计算只能使用该时刻已知的信息。具体体现为：

- 标准化/z-score 使用 expanding/rolling window
- 分位数分组基于当期截面或 expanding window 阈值
- 因子合成权重使用 rolling ICIR，而非全样本 ICIR
- ML 训练/验证按时序切分（walk-forward），禁止 shuffle
- 财务数据使用最新已披露的报告期数据，而非报告期数据（考虑披露延迟）

### 收益率分析规范

绩效分析模块必须输出：

- 收益分布：Mean、Median、P5、P25、P40、P75、P95（缺一不可）
- OLS t 检验 和 Newey-West 自相关稳健 t 检验，滞后阶数 $4 \times (T/100)^{2/9}$
- 经自相关调整的年化夏普比率（Lo 方法）

---

## 初始化流程

### 第一步：问清楚策略意图

在生成任何代码之前，与用户确认以下要素：

| 问题 | 选项/示例 |
|------|----------|
| 策略类型 | 横截面选股 / 时序择时 / 统计套利 / 事件驱动 / CTA趋势 / ML驱动 |
| Alpha 来源 | 基本面因子 / 技术指标 / 资金流 / 另类数据 / 统计关系 |
| 调仓频率 | 日度 / 周度 / 月度 / 季度 |
| 股票池 | 全A股 / 沪深300成分 / 中证500成分 / 某行业 / 自定义 |
| 基准 | 沪深300 / 中证500 / 中证全指 / 无（绝对收益） |
| 做空 | 仅做多 / 多空 |
| 项目名称 | 英文名，用作目录名和包名（如 `momentum_selector`） |
| 输出目录 | 默认为当前工作目录下 |

如果用户表达模糊，"帮我搭一个多因子选股框架"而没有具体说明，先确认上述要素。不要猜。

### 第二步：生成项目目录结构

确认后，在输出目录下创建以下结构：

```
{project_name}/
├── config.py                  # 策略配置（dataclass）
├── data/
│   ├── __init__.py
│   └── engine.py              # 数据引擎（加载、PIT 切片）
├── factors/
│   ├── __init__.py
│   └── builder.py             # 因子构造（PIT 标准化、缩尾、合成）
├── strategy/
│   ├── __init__.py
│   └── base.py                # 策略基类 + 用户策略模板
├── backtest/
│   ├── __init__.py
│   ├── engine.py              # 回测主循环
│   ├── portfolio.py           # 持仓管理
│   ├── execution.py           # 成交模拟
│   └── risk.py                # 风控
├── analysis/
│   ├── __init__.py
│   ├── metrics.py             # 绩效指标（含 NW 调整）
│   └── visualizer.py          # 可视化
├── main.py                    # 入口脚本
├── requirements.txt
├── CLAUDE.md                  # Claude Code 项目记忆文件
├── README.md                  # 项目文档模板（Markdown）
└── sub-agent/                 # 子 Agent 配置文件（运行时依赖）
```

### 第二步附：复制子 Agent 配置文件

生成目录结构后，将 skill 目录下的子 Agent 配置文件复制到项目的 `sub-agent/` 目录中：

```
{project_name}/sub-agent/
├── lookahead-auditor.md     ← 从 skills/quant-strategy/references/ 复制
└── dev-logger.md            ← 从 skills/quant-strategy/references/ 复制
```

这样项目运行时无需引用外部 skill 路径，Agent 指令随项目一起存放。

### 第三步：逐个模块写代码

按以下顺序生成各模块的 skeleton 代码。每个模块都要包含完整的接口定义、类型标注和 docstring。核心算法的默认实现（如 NW t 检验、PIT 标准化）直接写入，策略相关的方法留空（`raise NotImplementedError`）。

#### 1. `config.py` — 策略配置

用 dataclass 集中管理所有参数：

```python
@dataclass
class StrategyConfig:
    # 时间
    start_date: str = "2015-01-01"
    end_date: str = "2025-12-31"
    # 股票池
    universe: str = "000300.SH"           # 指数代码 or "all"
    # 调仓
    rebalance_freq: str = "M"              # D/W/M/Q
    # 基准
    benchmark: str = "000300.SH"
    # 费率
    commission_rate: float = 0.0003       # 双边佣金
    stamp_tax: float = 0.0005             # 卖出印花税
    slippage_bp: float = 1.0              # 滑点(bp)
    execution_price: str = "next_open"    # next_open / close / vwap
    # 风控
    max_position_pct: float = 0.10
    max_industry_pct: float = 0.30
    # 无风险利率
    risk_free_rate: float = 0.02
```

#### 2. `data/engine.py` — 数据引擎

核心类 `DataEngine`：

- `load_price(ts_codes)` — 通过 tushare MCP 拉取复权行情
- `load_financials(ts_codes, fields)` — 拉取财务数据，处理披露延迟
- `load_index_members(index_code)` — 拉取指数成分股
- `load_trading_calendar()` — 拉取交易日历
- `get_snapshot(date)` — 返回 `date` 日所有已知数据的 PIT 切片（核心方法）
- `get_trading_dates()` — 返回交易日列表

关键实现要点：
- 价格数据用后复权（`adj_factor` 修正）
- 年报披露截止 4/30，中报 8/31，季报在报告期结束后 1 个月内
- `get_snapshot(date)` 只能返回 `date` 及之前可用的数据

#### 3. `factors/builder.py` — 因子构造

提供 PIT 安全的因子计算工具函数：

- `zscore_pit(df, window=252)` — PIT z-score
- `winsorize_pit(df, lower=0.01, upper=0.99, window=252)` — PIT 缩尾
- `quantile_group_pit(factor, n_groups=5)` — PIT 分位数分组
- `compute_ic(factor_df, return_df, method='rank')` — 截面 IC 计算
- `compute_icir(ic_series, window=12)` — rolling ICIR
- `composite_factor(factors, weights=None)` — 因子合成（等权或 ICIR 加权）

因子定义示例（要留扩展空间）：

```python
# 常见因子模板，用户可自行添加
def momentum(df_price, n_months=12, skip_months=1):
    """过去 N 月收益率，跳过最近 M 月。"""
    ...

def volatility(returns, window=60):
    """历史波动率（年化）。"""
    ...
```

#### 4. `strategy/base.py` — 策略基类

```python
class BaseStrategy(ABC):
    """策略抽象基类。
    
    用户需继承此类并实现 generate_signal 方法。
    """
    
    @abstractmethod
    def generate_signal(self, date: pd.Timestamp,
                        data: Dict[str, pd.DataFrame]) -> pd.Series:
        """在 date 日生成下一期交易信号。
        
        此方法只能使用 data 中提供的数据（即 date 及之前已知的信息）。
        
        Args:
            date: 当前调仓日。
            data: DataEngine.get_snapshot(date) 返回的 PIT 数据。
        
        Returns:
            Series(index=ts_code, values=weight)，正为做多，负为做空。
            权重之和等于目标敞口。不在股票池内的股票返回 NaN。
        """
        ...
```

同时生成一个示例策略 `ExampleStrategy`（空壳，docstring 说明如何填充），让用户直观看到继承方式。

#### 5. `backtest/engine.py` — 回测主循环

事件驱动回测循环：

```python
class BacktestEngine:
    def __init__(self, config, data_engine, strategy, 
                 portfolio, execution_sim, risk_manager, analyzer):
        ...
    
    def run(self) -> pd.DataFrame:
        """执行回测主循环，返回日频净值序列。"""
        for date in self.trading_dates:
            self.portfolio.mark_to_market(date)
            if self.portfolio.is_rebalance_date(date):
                snapshot = self.data.get_snapshot(date)
                signals = self.strategy.generate_signal(date, snapshot)
                signals = self.risk_manager.check(signals, self.portfolio, date)
                target_weights = self.portfolio.compute_weights(signals, date)
                orders = self.portfolio.generate_orders(target_weights, date)
                trades = self.execution_sim.execute(orders, date)
                self.portfolio.update(trades, date)
            self.portfolio.record_daily(date)
        return self.portfolio.get_nav_series()
```

#### 6. `backtest/portfolio.py` — 持仓管理

- `mark_to_market(date)` — 按收盘价市值计价
- `compute_weights(signals, date)` — 信号转目标权重（含归一化、行业/个股权重上限）
- `generate_orders(target_weights, date)` — 权重差转订单列表
- `update(trades, date)` — 更新持仓和现金
- `record_daily(date)` — 记录每日快照
- `get_nav_series()` — 返回净值序列

#### 7. `backtest/execution.py` — 成交模拟

- 执行价格：`next_open`（T+1 开盘）或用户配置
- 涨跌停：涨停不买、跌停不卖（通过 `limit_list` 或价格比较判断）
- 停牌：成交量 = 0 不交易
- 流动性约束：单只股票日交易量不超过日均成交量的 10%
- 费用计算：佣金 + 卖出印花税 + 滑点

#### 8. `backtest/risk.py` — 风控

- 个股权重上限检查
- 行业暴露上限检查
- 最大换手率控制
- 个股止损（标记后下一期卖出）
- 组合最大回撤止损（触发后停止交易或减仓）

#### 9. `analysis/metrics.py` — 绩效指标（重点模块）

`PerformanceAnalyzer` 类，`compute_all_metrics()` 必须返回：

```python
{
    # 收益
    "annual_return": ..., "cumulative_return": ...,
    # 收益分布（缺一不可）
    "mean_return": ..., "median_return": ...,
    "p5": ..., "p25": ..., "p40": ..., "p75": ..., "p95": ...,
    "skewness": ..., "kurtosis": ...,
    # 风险
    "annual_volatility": ..., "max_drawdown": ...,
    "var_95": ..., "cvar_95": ...,
    # 风险调整
    "sharpe_ratio": ..., "sharpe_nw_adjusted": ...,
    "sortino_ratio": ..., "calmar_ratio": ...,
    "information_ratio": ...,
    # 显著性
    "alpha": ..., "t_ols": ..., "p_ols": ...,
    "t_nw": ..., "p_nw": ..., "nw_lags": ...,
    # 交易
    "win_rate": ..., "profit_loss_ratio": ...,
    "annual_turnover": ...,
}
```

Newey-West t 检验用 `statsmodels.stats.sandwich_covariance.cov_hac`，滞后阶数按 Newey-West 自动公式计算。

NW 调整夏普比率：$Sharpe_{NW} = Sharpe / \sqrt{1 + 2 \sum_{k=1}^{L} w(k, L) \rho_k}$，Bartlett 权重 $w(k, L) = 1 - k/(L+1)$。

#### 10. `analysis/visualizer.py` — 可视化

生成 5 张必选图 + 可选图：

**必选**：
1. 净值曲线（对数坐标，策略 vs 基准）
2. 回撤曲线（策略 vs 基准）
3. 月度收益热力图
4. 收益分布直方图（叠加正态拟合，标注均值和中位数）
5. 滚动 12 月夏普比率

**可选**：
- 滚动最大回撤
- 年度收益柱状图
- 因子暴露时序图

#### 11. `main.py` — 入口脚本

```python
"""项目入口。调整策略逻辑后运行此脚本进行回测。"""

from config import StrategyConfig
from data.engine import DataEngine
from strategy.base import ExampleStrategy
from backtest.engine import BacktestEngine
from backtest.portfolio import PortfolioManager
from backtest.execution import ExecutionSimulator, ExecutionConfig
from backtest.risk import RiskManager, RiskConfig
from analysis.metrics import PerformanceAnalyzer
from analysis.visualizer import Visualizer

def main():
    config = StrategyConfig()
    # 1. 加载数据
    data = DataEngine(config)
    data.load_all()
    # 2. 初始化策略（替换为你的策略）
    strategy = ExampleStrategy(config)
    # 3. 初始化回测
    portfolio = PortfolioManager(config)
    execution = ExecutionSimulator(ExecutionConfig())
    risk = RiskManager(RiskConfig())
    engine = BacktestEngine(config, data, strategy, portfolio, execution, risk)
    # 4. 回测
    nav = engine.run()
    # 5. 绩效分析
    analyzer = PerformanceAnalyzer(nav, config)
    metrics = analyzer.compute_all_metrics()
    analyzer.print_report(metrics)
    # 6. 可视化
    viz = Visualizer(nav, config)
    viz.plot_all()
    viz.show()

if __name__ == "__main__":
    main()
```

#### 12. `requirements.txt`

```
numpy>=1.24
pandas>=2.0
scipy>=1.10
statsmodels>=0.14
matplotlib>=3.7
seaborn>=0.12
```

#### 13. `CLAUDE.md` — 项目记忆文件

读取 `references/CLAUDE-template.md`，将其中的 `{...}` 占位符替换为用户确认的实际内容，然后写入项目的 `CLAUDE.md`。模板包含完整的核心设计原则、子 Agent 使用规范和开发流程，不得省略任何章节。

#### 14. `README.md` — 项目文档模板

按以下结构生成。注意不要与 CLAUDE.md 重复（CLAUDE.md 面向 Claude，README 面向人类读者）：

```markdown
# {项目名称}

> 由 Claude Code quant-strategy skill 自动生成的量化策略回测项目。

## 策略概要
- 类型：{来自用户确认}
- Alpha 来源：{来自用户确认}
- 调仓：{来自用户确认}
- 股票池：{来自用户确认}
- 基准：{来自用户确认}

## 快速开始
1. 安装依赖：`pip install -r requirements.txt`
2. 编写策略：编辑 `strategy/base.py`，继承 `BaseStrategy` 实现 `generate_signal()`
3. 修改配置：编辑 `config.py` 调整参数
4. 运行回测：`python main.py`

## 项目结构
{项目目录树}

## 规则说明
本项目强制遵守两条核心原则，详见 `CLAUDE.md`：
1. **禁止引入未来信息** — 所有计算仅使用当前已知数据
2. **收益分析规范** — 返回分布统计 + NW t 检验 + NW 调整夏普

## 结果
{待补充回测结果摘要}

## 风险提示
量化策略存在过拟合风险，历史回测不代表未来表现。实盘前请充分验证。
```

---

## 生成代码后的说明

所有文件生成完毕后，必须向用户说明：

1. **入口在哪里**：`main.py` 是主入口，运行即可执行完整流程
2. **策略写在哪**：`strategy/base.py` 中的策略类，只需填充 `generate_signal` 方法
3. **配置怎么改**：编辑 `config.py` 中的 `StrategyConfig` 参数
4. **结果怎么看**：控制台输出绩效表 + 弹出图表窗口
5. **关键约束**：再次强调不能引入未来信息，因子计算和绩效分析的 PIT 原则
6. **下一步**：从 tushare MCP 拉取真实数据替换 `data/engine.py` 中的 mock 实现

全部代码生成后，**立即调用 look-ahead 审计 Agent**（`references/lookahead-auditor.md`）审查所有生成的 `.py` 文件。若审计发现违规，必须修复后向用户报告修复结果。

---

## 参考文件

项目生成时，对策略细节有疑问可查阅以下参考文件：
- `references/strategy-types.md` — 策略分类详解
- `references/factor-library.md` — 因子定义与 PIT 实现模板
- `references/backtesting-design.md` — 回测框架详细设计
- `references/performance-metrics.md` — 绩效指标公式与 NW 调整方法
- `references/lookahead-auditor.md` — Look-ahead 审计 Agent 指令，审查 7 类前视偏差
- `references/dev-logger.md` — 开发日志 Agent 指令，记录每次改动（精确到分钟）
