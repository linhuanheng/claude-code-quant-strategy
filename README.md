# Quant Strategy Skill for Claude Code

一个面向量化交易策略开发者的 [Claude Code](https://claude.ai/code) Skill。帮助用户一键搭建量化策略项目的初始代码框架，内置严格的**前视偏差审计**和**收益分析规范**。

## 核心特性

- **项目脚手架生成**：一条指令生成完整的量化策略项目骨架（14 个文件，覆盖数据引擎、因子构造、回测引擎、绩效分析、可视化）
- **禁止未来信息（铁律）**：所有模块接口设计遵循 point-in-time 原则，确保回测结果可实盘复现
- **内置 Look-ahead 审计 Agent**：独立的子 Agent，在展示分析结果前自动审查 7 类前视偏差，发现违规必须修复
- **开发日志 Agent**：自动记录每次代码改动的细节，精确到分钟
- **完整的收益分析规范**：7 种分位数 + Newey-West 自相关调整 t 检验 + 经调整的年化夏普比率
- **支持 6 类策略**：横截面选股、时序择时、统计套利、事件驱动、CTA 趋势、ML 驱动

## 项目结构

### Skill 本身

```
quant-strategy/
├── SKILL.md                                    # 主 Skill 定义
└── references/
    ├── strategy-types.md                       # 策略分类与选型决策树
    ├── factor-library.md                       # 因子库与 PIT 实现模板
    ├── backtesting-design.md                   # 回测框架架构设计
    ├── performance-metrics.md                  # 绩效指标详解（含 NW 调整）
    ├── lookahead-auditor.md                    # 前视偏差审计 Agent
    └── dev-logger.md                           # 开发日志记录 Agent
```

### 生成的策略项目

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
│   ├── portfolio.py           # 持仓管理
│   ├── execution.py           # 成交模拟
│   └── risk.py                # 风控
├── analysis/
│   ├── metrics.py             # 绩效指标（含 NW 调整）
│   └── visualizer.py          # 可视化
├── sub-agent/
│   ├── lookahead-auditor.md   # 审计 Agent（前置关卡）
│   └── dev-logger.md          # 日志 Agent（全程记录）
├── main.py                    # 入口脚本
├── requirements.txt
├── CLAUDE.md                  # 项目记忆（规则自动加载）
└── README.md                  # 项目文档
```

## 快速开始

### 安装

在 Claude Code 中安装此 Skill（`.skill` 文件）：

```bash
# 打包为 .skill 文件
pip install -r requirements.txt  # 如果打包脚本有依赖
python -m scripts.package_skill /path/to/quant-strategy

# 或直接复制到 skills 目录
cp -r quant-strategy ~/.claude/skills/
```

### 使用

在 Claude Code 对话中输入：

```
帮我创建一个多因子选股策略项目
```

Skill 会自动：
1. 与你确认策略类型、alpha 来源、调仓频率、股票池、基准
2. 生成完整的项目目录和 skeleton 代码
3. 将审计 Agent 和日志 Agent 配置复制到项目的 `sub-agent/` 目录
4. 生成 `CLAUDE.md` 确保后续协作遵循项目规则
5. 自动运行 Look-ahead 审计确保初始代码无前视偏差

## 两个子 Agent

### Look-ahead 审计 Agent（前置关卡）

- **触发**：用户每次请求收益分析时自动启动
- **职责**：审查 7 类前视偏差（全样本标准化、全样本分位数、财务披露延迟、T日收盘决策、ML数据泄漏、全样本因子筛选、ST/退市信息滥用）
- **规则**：审计通过才展示分析结果，未通过必须先修复

### 开发日志 Agent（全程记录）

- **触发**：任何代码修改、参数调整、回测运行、审计完成后自动记录
- **输出**：追加到项目 `DEV_LOG.md`，精确到分钟
- **格式**：序号 + 时间戳 + 改动类型 + 涉及文件 + 改动详情 + 结果

## 收益分析规范

每次策略评估必须输出：

| 类别 | 指标 |
|------|------|
| 收益分布 | Mean、Median、P5、P25、P40、P75、P95 |
| 显著性 | OLS t 值 + Newey-West 调整 t 值 + p 值 |
| 风险调整 | 夏普比率 + NW 调整夏普（Lo 方法） |
| 风险 | 波动率、最大回撤、VaR、CVaR |
| 交易 | 胜率、盈亏比、年化换手率 |

## 数据来源

支持通过 [Tushare MCP](https://tushare.pro/) 获取 A 股数据。可覆盖：
- 行情数据（日线、分钟、复权）
- 财务数据（三大表 + 财务指标）
- 估值数据（因子、指数成分）
- 资金流（北向资金、融资融券、龙虎榜）
- 事件数据（业绩预告、分红、回购、解禁）

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
