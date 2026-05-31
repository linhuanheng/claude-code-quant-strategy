# Quant Strategy Skill for Claude Code

一个面向量化交易策略开发者的 [Claude Code](https://claude.ai/code) Skill，帮助用户快速搭建量化策略项目的初始代码框架。内置**策略代码审计 Agent**（前视偏差 + 实盘约束）和**开发日志 Agent**（全程记录）。

## 核心特性

- **项目脚手架一键生成**：一条指令生成完整的量化策略项目骨架（14 个文件），覆盖数据引擎、因子构造、回测引擎、绩效分析、可视化
- **策略代码审计 Agent**：双重前置关卡——代码改动后必须审计通过才能回测，展示收益前必须审计通过。检查 7 类前视偏差 + 5 项实盘交易约束（最小交易单位、做空限制、T+1、涨跌停、资金约束）
- **开发日志 Agent**：每次改动自动记录到 `DEV_LOG.md`，精确到分钟。顶部维护文件追踪表，记录每个文件的最后修改时间
- **禁止未来信息（铁律）**：所有模块接口遵循 point-in-time 原则，确保回测结果可实盘复现
- **完整收益分析规范**：7 种分位数 + OLS/Newey-West 双 t 检验 + NW 调整年化夏普
- **支持 6 类策略**：横截面选股、时序择时、统计套利、事件驱动、CTA 趋势、ML 驱动

## Skill 结构

```
quant-strategy/
├── SKILL.md                                    # 主 Skill 定义
└── references/
    ├── CLAUDE-template.md                      # 生成的 CLAUDE.md 模板
    ├── strategy-types.md                       # 策略分类与选型决策树
    ├── factor-library.md                       # 因子库与 PIT 实现模板
    ├── backtesting-design.md                   # 回测框架架构设计
    ├── performance-metrics.md                  # 绩效指标详解（含 NW 调整）
    ├── lookahead-auditor.md                    # 策略代码审计 Agent
    └── dev-logger.md                           # 开发日志记录 Agent
```

## 生成的策略项目

```
{project_name}/
├── config.py                  # 策略配置（dataclass）
├── data/
│   └── engine.py              # 数据引擎（加载、PIT 切片）
├── factors/
│   └── builder.py             # 因子构造（PIT 标准化、缩尾、合成）
├── strategy/
│   └── base.py                # 策略基类 + 用户策略模板
├── backtest/
│   ├── engine.py              # 回测主循环
│   ├── portfolio.py           # 持仓管理（含 T+1、整手约束）
│   ├── execution.py           # 成交模拟（含涨跌停、滑点）
│   └── risk.py                # 风控
├── analysis/
│   ├── metrics.py             # 绩效指标（含 NW 调整）
│   └── visualizer.py          # 可视化（5 张必选图）
├── sub-agent/
│   ├── lookahead-auditor.md   # 审计 Agent 指令
│   └── dev-logger.md          # 日志 Agent 指令
├── main.py                    # 入口脚本
├── requirements.txt
├── CLAUDE.md                  # 项目记忆（规则自动加载）
├── DEV_LOG.md                 # 开发日志（含文件追踪表）
└── README.md                  # 项目文档
```

## 快速开始

### 安装

```bash
cp -r quant-strategy ~/.claude/skills/
```

### 使用

在 Claude Code 对话中输入：

```
帮我创建一个多因子选股策略项目
```

Skill 会自动：
1. 确认策略类型、alpha 来源、调仓频率、股票池、基准
2. 生成完整的项目目录和 skeleton 代码
3. 将审计 Agent 和日志 Agent 配置复制到 `sub-agent/` 目录
4. 生成 `CLAUDE.md`（含完整规则，Claude Code 自动加载）
5. 自动运行审计 Agent 确保初始代码无违规

## 核心原则

### 1. 禁止引入未来信息

- 因子标准化：必须使用 rolling/expanding window，禁止全样本均值/标准差
- 分位数分组：每期截面独立计算，禁止全样本排序分组
- 财务数据：年报 +4 月、中报 +2 月、季报 +1 月后才视为可用
- 信号执行：T 日收盘信号 → T+1 日开盘执行
- ML 模型：walk-forward，禁止 shuffle 或随机交叉验证

### 2. 收益分析规范

| 类别 | 指标 |
|------|------|
| 收益分布 | Mean、Median、P5、P25、P40、P75、P95 |
| 显著性 | OLS t 值 + Newey-West 调整 t 值 + p 值（\*\*\*/\*\*/ \* ） |
| 风险调整 | 夏普比率 + NW 调整夏普（Lo 2002） |
| 风险 | 波动率、最大回撤、VaR、CVaR |
| 交易 | 胜率、盈亏比、年化换手率 |

## 子 Agent 详解

### Agent A：策略代码审计 Agent

双重前置关卡——不通过审计，不回测，不出结果。

**审计范围**：

| 类别 | 检查项 | 严重 |
|------|--------|------|
| 前视偏差 | 全样本标准化 | 高 |
| 前视偏差 | 全样本分位数分组 | 高 |
| 前视偏差 | 财务数据披露延迟忽略 | 中 |
| 前视偏差 | T 日收盘价用于 T 日决策 | 中 |
| 前视偏差 | ML 训练数据泄漏 | 高 |
| 前视偏差 | 因子筛选使用全样本统计量 | 高 |
| 前视偏差 | 停牌/退市/ST 信息前视 | 低-中 |
| 实盘约束 | 最小交易单位（一手 100 股） | 高 |
| 实盘约束 | 做空限制（融券标的、成本） | 高 |
| 实盘约束 | T+1 交易制度 | 高 |
| 实盘约束 | 涨跌停可交易性 | 中 |
| 实盘约束 | 资金与持仓约束 | 高 |

### Agent B：开发日志 Agent

全程记录开发过程，每次改动自动追加到 `DEV_LOG.md`。

- 每次代码修改、参数调整、回测运行、审计完成后自动触发
- 记录格式：序号 + 时间戳（精确到分钟）+ 改动类型 + 涉及文件 + 详情 + 结果
- 顶部维护文件追踪表，记录每个文件的最后修改时间和最近改动摘要

## 数据来源

支持通过 [Tushare MCP](https://tushare.pro/) 获取 A 股数据，覆盖行情、财务、估值、资金流、事件数据。

## 依赖

- Python ≥ 3.8
- pandas, numpy, scipy
- statsmodels ≥ 0.14（Newey-West HAC）
- matplotlib, seaborn（可视化）
- [Tushare](https://tushare.pro/)（数据获取，可选）

## 许可证

MIT License

## 声明

本 Skill 仅供研究和学习使用。量化交易存在风险，任何基于此 Skill 生成的策略在实盘前请充分验证和测试。不构成任何投资建议。
