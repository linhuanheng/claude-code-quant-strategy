# 因子库与 PIT 构造模板

## PIT 构造核心规则

每个因子在各时间点 `t` 的计算，只能使用 `t` 日已知的数据。具体而言：

```
因子值_t = f(数据_t, 数据_{t-1}, ..., 数据_{t-N})
         ≠ f(数据_0, 数据_1, ..., 数据_T)  // 禁止全样本统计
```

### 标准化（Z-score）的 PIT 实现

```python
def zscore_pit(df: pd.DataFrame, window: int = 252) -> pd.DataFrame:
    """PIT 模式的截面 z-score 标准化。
    
    df: index=date, columns=ts_code, values=raw_factor
    对每个交易日，使用过去 window 日的均值和标准差做标准化。
    
    Returns: 标准化后的因子值 (index=date, columns=ts_code)
    """
    rolling_mean = df.rolling(window=window, min_periods=min(window, 63)).mean()
    rolling_std = df.rolling(window=window, min_periods=min(window, 63)).std()
    return (df - rolling_mean) / rolling_std.replace(0, np.nan)
```

### 分位数分组的 PIT 实现

```python
def quantile_group_pit(factor: pd.DataFrame, n_groups: int = 5) -> pd.DataFrame:
    """PIT 模式的分位数分组。
    
    每个交易日截面单独计算分位数分组，不依赖未来数据。
    
    Returns: 分组标签 0..n_groups-1 (index=date, columns=ts_code)
    """
    return factor.apply(
        lambda row: pd.qcut(row, n_groups, labels=False, duplicates='drop'),
        axis=1
    )
```

### 极值缩尾的 PIT 实现

```python
def winsorize_pit(factor: pd.DataFrame, 
                  lower: float = 0.01, 
                  upper: float = 0.99,
                  window: int = 252) -> pd.DataFrame:
    """PIT 模式的截面缩尾处理。
    
    使用 expanding window 计算每个截面的缩尾阈值，
    不能使用全样本分位数。
    """
    result = factor.copy()
    for t_idx in range(window, len(factor)):
        hist = factor.iloc[:t_idx+1]
        low_val = hist.quantile(lower, axis=1).iloc[-1]
        high_val = hist.quantile(upper, axis=1).iloc[-1]
        row = result.iloc[t_idx]
        result.iloc[t_idx] = row.clip(lower=low_val, upper=high_val)
    return result
```

---

## 常见因子定义

### 一、价值因子

#### EP (Earnings-to-Price)
```
EP = 净利润(TTM) / 总市值
```
- 财务数据使用最新披露的合并报表数据
- 净利润 TTM = 最近 4 个单季度净利润之和
- **PIT 注意**：报告披露有延迟，在 4 月底之前可能年报尚未披露，此时用最近可用的数据

#### BP (Book-to-Price)
```
BP = 归属于母公司股东权益 / 总市值
```

#### SP (Sales-to-Price)
```
SP = 营业收入(TTM) / 总市值
```

#### CFP (Cash-Flow-to-Price)
```
CFP = 经营活动现金流(TTM) / 总市值
```

#### DP (Dividend Yield)
```
DP = 近 12 月每股现金分红 / 股价
```

### 二、动量因子

#### 过去 N 月收益率（Skip 1M）

```python
def momentum(df_price: pd.DataFrame, n_months: int = 12,
             skip_months: int = 1) -> pd.DataFrame:
    """计算过去 N 月收益率，跳过最近 M 月。
    
    跳过最近 1 个月是为了避免短期反转效应的干扰。
    使用复权收盘价计算。
    
    df_price: index=date, columns=ts_code, values=adj_close
    """
    n_days = n_months * 21
    skip_days = skip_months * 21
    # T-skip_days 到 T-skip_days-n_days 的收益
    ret = df_price.shift(skip_days) / df_price.shift(skip_days + n_days) - 1
    return ret
```

#### 均线乖离率
```
BIAS_N = (收盘价 - MA_N) / MA_N
```
N 可取 20/60/120 日，反映短期/中期/长期趋势强度。

#### RSI (Relative Strength Index)
```
RSI_14 = 100 - 100/(1 + RS), RS = Avg_Gain_14 / Avg_Loss_14
```
注意：RSI 需要 14 日以上数据才能计算，初始值用 SMA 而非 EMA。

### 三、质量因子

#### ROE
```
ROE = 净利润(TTM) / 平均归属于母公司股东权益
```

#### 毛利率
```
Gross_Margin = (营业收入 - 营业成本) / 营业收入
```

#### 应计项目 (Accruals)

这是学术界非常重视的因子，应计越高盈利质量越差：

```
Accruals = (Δ流动资产 - Δ现金) - (Δ流动负债 - Δ短期借款 - Δ应交税费) - 折旧摊销
Scaled_Accruals = Accruals / 平均总资产
```

负的 Scaled Accruals 意味着盈利质量高（现金含量高）。

#### 资产周转率
```
Asset_Turnover = 营业收入(TTM) / 平均总资产
```

### 四、波动率与风险因子

#### 历史波动率

```python
def historical_volatility(returns: pd.DataFrame,
                          window: int = 60) -> pd.DataFrame:
    """计算历史波动率（年化）。
    
    使用简单收益率而非对数收益率，
    窗口内至少需要 30 个有效交易日。
    """
    vol = returns.rolling(window=window, min_periods=30).std() * np.sqrt(252)
    return vol
```

#### 特质波动率 (Idiosyncratic Volatility)

对 Fama-French 三因子或 CAPM 回归的残差波动率：

```python
def idiosyncratic_vol(returns: pd.DataFrame,
                      mkt_returns: pd.Series,
                      window: int = 252,
                      min_periods: int = 120) -> pd.Series:
    """计算每只股票的特质波动率。
    
    PIT 实现：对每只股票、每个时间点，
    使用 [t-window, t] 区间的数据做回归。
    """
    # 滚动 OLS: ret_i ~ alpha + beta * mkt_ret + epsilon
    # 特质波动率 = std(epsilon_i) * sqrt(252)
    ...
```

### 五、成长因子

#### 营收增速
```
Rev_Growth_YoY = (营业收入_TTM - 营业收入_TTM_LY) / |营业收入_TTM_LY|
Rev_Growth_QoQ = (营业收入_Q - 营业收入_Q_LQ) / |营业收入_Q_LQ|
```

#### 净利润增速（同理）

#### 分析师预期上调比例

如果数据可用，使用一致预期的变化。

### 六、资金流/情绪因子

#### 北向资金净流入
```
North_Flow_Ratio = 近 N 日北向净买入金额 / 近 N 日成交额
```

#### 融资买入占比
```
Margin_Buy_Ratio = 融资买入额 / 总成交额
```

#### 小额资金流（散户 vs 机构）
基于 `moneyflow` 的小单/中单/大单/特大单分类。

### 七、技术指标因子

| 指标 | 计算 | 含义 |
|------|------|------|
| MACD | EMA12 - EMA26, Signal=EMA9(MACD) | 趋势/背离 |
| RSI | 见上文 | 超买超卖 |
| ATR | EMA(TR, 14) | 波动率 |
| OBV | 量价累积 | 量价关系 |
| 布林带位置 | (Price-Lower)/(Upper-Lower) | 价格相对位置 |

**PIT 警告**：技术指标的参数不能通过全样本优化确定！如需参数优化，必须在训练集上完成。

---

## 因子合成方法

### 等权合成
```
Composite = (1/N) * Σ(Factor_i_zscore)
```

### ICIR 加权
```
Weight_i = ICIR_i / Σ|ICIR_j|
Composite = Σ(Weight_i * Factor_i_zscore)
```
**PIT 注意**：ICIR 必须用滚动窗口（如过去 12 期）计算，不能用全样本 ICIR。

### 信息率加权
```
Weight_i = max(ICIR_i, 0) / Σ max(ICIR_j, 0)
```
类似 ICIR 加权但只保留正贡献因子。

---

## 因子检验清单

生成因子后，按以下顺序检验：

1. **覆盖度**：每期有多少股票有有效因子值？太低说明数据问题。
2. **IC 序列**：rolling Rank IC，汇报均值、标准差、ICIR、IC>0 比例。
3. **IC 衰减**：因子对未来 1/2/3/...期收益的 IC 变化。
4. **分层单调性**：5 组或 10 组等权收益是否随因子值单调。
5. **多空收益**：top - bottom 组的收益分布和 NW t 值。
6. **换手率**：因子排名变化的程度，决定交易成本。
7. **与其他因子的相关性**：判断因子是否提供独立信息。
8. **不同市场环境**：牛熊市、大小盘风格下的因子表现差异。
