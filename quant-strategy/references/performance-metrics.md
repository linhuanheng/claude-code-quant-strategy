# 绩效指标详解

## 一、收益分布统计（必须全部汇报）

对于策略日/周/月收益率序列 $r_t$：

```python
def compute_return_distribution(returns: np.ndarray) -> dict:
    """计算收益率的完整分布统计。
    
    必须汇报以下所有指标，缺一不可。
    """
    return {
        "mean": np.mean(returns),
        "median": np.median(returns),
        "p5": np.percentile(returns, 5),
        "p25": np.percentile(returns, 25),
        "p40": np.percentile(returns, 40),
        "p75": np.percentile(returns, 75),
        "p95": np.percentile(returns, 95),
        "std": np.std(returns, ddof=1),
        "skewness": stats.skew(returns),
        "kurtosis": stats.kurtosis(returns),  # excess kurtosis
        "min": np.min(returns),
        "max": np.max(returns),
        "n_obs": len(returns),
    }
```

汇报模板：

```
              策略日收益    基准日收益    超额日收益
Mean          0.XX%       0.XX%        0.XX%
Median        0.XX%       0.XX%        0.XX%
P5           -X.XX%          -            -
P25          -X.XX%          -            -
P40          -0.XX%          -            -
P75           0.XX%          -            -
P95           0.XX%          -            -
Std           0.XX%       0.XX%        0.XX%
Skewness      0.XX       0.XX          0.XX
Kurtosis      0.XX       0.XX          0.XX
```

---

## 二、常规绩效指标

### 年化收益
$$r_{annual} = (1 + \bar{r})^{n} - 1$$

其中 $\bar{r}$ 为期间平均收益率，$n$ 为年化因子：
- 日频：252
- 周频：52
- 月频：12

### 年化波动率
$$\sigma_{annual} = \sigma_{period} \times \sqrt{n}$$

### 最大回撤 (Maximum Drawdown)
$$MDD = \max_{t} \left( \frac{\max_{\tau \leq t} NAV_\tau - NAV_t}{\max_{\tau \leq t} NAV_\tau} \right)$$

```python
def compute_max_drawdown(nav: np.ndarray) -> tuple:
    """计算最大回撤及起止日期。
    
    Returns:
        (mdd_pct, peak_date, trough_date, recovery_date)
    """
    peak = np.maximum.accumulate(nav)
    drawdown = (peak - nav) / peak
    mdd_idx = np.argmax(drawdown)
    trough_date = mdd_idx
    peak_date = np.argmax(nav[:mdd_idx + 1])
    return drawdown[mdd_idx], peak_date, trough_date, None
```

### 胜率
$$Win\_Rate = \frac{\sum \mathbb{1}[r_t > 0]}{\sum \mathbb{1}[r_t \neq 0]}$$

### 盈亏比
$$Profit\_Loss\_Ratio = \frac{mean(r_t | r_t > 0)}{-mean(r_t | r_t < 0)}$$

### VaR (Value at Risk)
$$VaR_\alpha = \text{Percentile}(r_t, 1-\alpha)$$

标准取 $\alpha = 0.05$（95% VaR）和 $\alpha = 0.01$（99% VaR）。

### CVaR (Expected Shortfall)
$$CVaR_\alpha = \mathbb{E}[r_t | r_t \leq VaR_\alpha]$$

---

## 三、风险调整收益指标

### Sharpe Ratio
$$Sharpe = \frac{\bar{r} - r_f}{\sigma_r} \times \sqrt{n}$$

其中 $r_f$ 为无风险利率（通常取 2%）。

### 自相关调整后的 Sharpe Ratio

收益率自相关会使传统夏普比率有偏。使用 **Andrews (1991)** 或 **Lo (2002)** 方法调整：

```python
def sharpe_nw_adjusted(returns: np.ndarray, 
                       rf_daily: float = 0.02 / 252,
                       n_periods: int = 252) -> float:
    """经序列相关调整的夏普比率。
    
    Lo (2002) 方法：
    Sharpe_adj = Sharpe / sqrt(1 + 2 * Σ ρ_k)
    其中 ρ_k 为收益率的 k 阶自相关系数。
    """
    excess = returns - rf_daily
    mean_excess = np.mean(excess)
    std_excess = np.std(excess, ddof=1)
    
    # 计算自相关调整因子
    n = len(returns)
    max_lag = int(4 * (n / 100) ** (2 / 9))
    rho_sum = 0.0
    for k in range(1, max_lag + 1):
        rho = np.corrcoef(excess[:-k], excess[k:])[0, 1]
        # Bartlett 权重
        weight = 1 - k / (max_lag + 1)
        rho_sum += weight * rho
    
    adjustment = np.sqrt(1 + 2 * rho_sum)
    sharpe = mean_excess / std_excess * np.sqrt(n_periods)
    return sharpe / adjustment
```

### Sortino Ratio
$$Sortino = \frac{\bar{r} - r_f}{\sigma_{down}} \times \sqrt{n}$$

其中 $\sigma_{down}$ 为下行标准差（只计算负收益的波动）：

$$\sigma_{down} = \sqrt{\frac{1}{T}\sum_t \min(r_t - r_f, 0)^2}$$

### Calmar Ratio
$$Calmar = \frac{r_{annual}}{|MDD|}$$

### Information Ratio（相对基准）
$$IR = \frac{\bar{r}_{excess}}{\sigma_{r_{excess}}} \times \sqrt{n}$$

其中 $r_{excess} = r_{strategy} - r_{benchmark}$。

---

## 四、显著性检验

### OLS t 检验

$$t_{OLS} = \frac{\bar{r}}{SE(r)} = \frac{\bar{r}}{\sigma_r / \sqrt{T}}$$

### Newey-West 自相关稳健 t 检验

收益率存在自相关时，OLS 标准误会低估，导致 t 值偏高（假阳性）。**必须**使用 Newey-West 调整：

$$Var_{NW}(\bar{r}) = \frac{1}{T} \left[ \hat{\gamma}_0 + 2 \sum_{k=1}^{L} w(k, L) \hat{\gamma}_k \right]$$

其中：
- $\hat{\gamma}_k = \frac{1}{T} \sum_{t=k+1}^{T} (r_t - \bar{r})(r_{t-k} - \bar{r})$
- $w(k, L) = 1 - \frac{k}{L+1}$（Bartlett 核）
- $L = \lfloor 4 \times (T/100)^{2/9} \rfloor$（Newey-West 自动滞后选择）

$$t_{NW} = \frac{\bar{r}}{\sqrt{Var_{NW}(\bar{r})}}$$

```python
from statsmodels.stats.sandwich_covariance import cov_hac

def newey_west_t_stat(returns: np.ndarray) -> dict:
    """计算 OLS 和 NW 调整后的 t 统计量。
    
    Args:
        returns: 收益率序列（超额收益或策略收益）。
    
    Returns:
        dict with t_ols, p_ols, t_nw, p_nw, nw_lags
    """
    from scipy import stats as scipy_stats
    from statsmodels.regression.linear_model import OLS
    import numpy as np
    
    n = len(returns)
    max_lags = int(4 * (n / 100) ** (2 / 9))
    
    # OLS: r_t = alpha + epsilon_t (即截距项回归)
    X = np.ones((n, 1))
    model = OLS(returns, X).fit()
    alpha = model.params[0]
    
    # OLS standard error
    se_ols = model.bse[0]
    t_ols = alpha / se_ols
    p_ols = 2 * (1 - scipy_stats.t.cdf(abs(t_ols), n - 1))
    
    # Newey-West HAC standard error
    cov_hac_matrix = cov_hac(model, nlags=max_lags, kernel='bartlett')
    se_nw = np.sqrt(cov_hac_matrix[0, 0])
    t_nw = alpha / se_nw
    p_nw = 2 * (1 - scipy_stats.t.cdf(abs(t_nw), n - 1))
    
    return {
        "alpha": alpha,
        "t_ols": t_ols,
        "p_ols": p_ols,
        "t_nw": t_nw,
        "p_nw": p_nw,
        "nw_lags": max_lags,
    }
```

### 显著性水平标注

| p 值范围 | 标注 |
|---------|------|
| p < 0.01 | \*\*\* |
| p < 0.05 | \*\*  |
| p < 0.10 | \*   |
| p >= 0.10 | (不标注) |

### 汇报格式

```
超额收益显著性检验:
  Alpha (年化)        X.XX%
  OLS t-stat:         X.XX  (p = 0.XXX)
  NW t-stat:          X.XX  (p = 0.XXX) **
  NW lags:            N

因子 IC 显著性:
  Mean Rank IC:       0.0XX
  OLS t-stat:         X.XX
  NW t-stat:          X.XX  ***
  ICIR:               X.XX
```

---

## 五、归因分析

### Brinson 归因（适用于行业配置 + 选股）

将超额收益分解为：
- **配置效应**：行业超/低配带来的收益
- **选股效应**：行业内选股带来的收益
- **交互效应**：配置与选股的交乘项

$$R_{excess} = \sum_i (w_i^P - w_i^B) R_i^B + \sum_i w_i^B (R_i^P - R_i^B) + \sum_i (w_i^P - w_i^B)(R_i^P - R_i^B)$$

### 因子暴露归因

对策略收益做时间序列回归：
$$r_t^{strategy} = \alpha + \sum_{k} \beta_k \cdot Factor_{k,t} + \epsilon_t$$

汇报每个因子的暴露 $\beta_k$ 和 NW t 值，以及截距项 alpha 的 NW t 值（即因子调整后的纯 alpha）。

---

## 六、可视化清单

### 1. 净值曲线

```python
def plot_nav_curve(strategy_nav, benchmark_nav=None):
    fig, ax = plt.subplots(figsize=(12, 6))
    ax.semilogy(strategy_nav.index, strategy_nav.values, 
                label='Strategy', linewidth=1)
    if benchmark_nav is not None:
        ax.semilogy(benchmark_nav.index, benchmark_nav.values,
                    label='Benchmark', linewidth=1, alpha=0.7)
    ax.legend()
    ax.set_title('Cumulative NAV (log scale)')
    ax.set_ylabel('NAV')
    ax.grid(True, alpha=0.3)
```

### 2. 回撤曲线

```python
def plot_drawdown(nav):
    peak = nav.expanding().max()
    drawdown = (nav - peak) / peak * 100
    fig, ax = plt.subplots(figsize=(12, 4))
    ax.fill_between(drawdown.index, 0, drawdown.values, 
                    color='red', alpha=0.3)
    ax.plot(drawdown.index, drawdown.values, color='red', linewidth=0.5)
    ax.set_title('Drawdown')
    ax.set_ylabel('Drawdown (%)')
```

### 3. 月度收益热力图

```python
def plot_monthly_heatmap(daily_returns):
    monthly = daily_returns.resample('ME').apply(
        lambda x: (1 + x).prod() - 1
    )
    matrix = pd.DataFrame({
        'year': monthly.index.year,
        'month': monthly.index.month,
        'return': monthly.values * 100
    }).pivot(index='month', columns='year', values='return')
    
    fig, ax = plt.subplots(figsize=(14, 8))
    sns.heatmap(matrix, annot=True, fmt='.1f', cmap='RdYlGn',
                center=0, ax=ax, cbar_kws={'label': 'Return (%)'})
    ax.set_title('Monthly Returns Heatmap')
```

### 4. 收益分布图

```python
def plot_return_distribution(returns):
    fig, ax = plt.subplots(figsize=(10, 6))
    ax.hist(returns * 100, bins=50, density=True, alpha=0.6, label='Returns')
    x = np.linspace(returns.min(), returns.max(), 200) * 100
    mu, sigma = returns.mean() * 100, returns.std(ddof=1) * 100
    ax.plot(x, stats.norm.pdf(x * 100, mu, sigma), 
            'r-', linewidth=1.5, label='Normal fit')
    ax.axvline(returns.mean() * 100, color='red', linestyle='--', 
               label=f'Mean: {mu:.2f}%')
    ax.axvline(np.median(returns) * 100, color='blue', linestyle='--',
               label=f'Median: {np.median(returns)*100:.2f}%')
    ax.legend()
    ax.set_xlabel('Return (%)')
    ax.set_ylabel('Density')
```

### 5. 滚动指标

```python
def plot_rolling_metrics(returns, window=252):
    fig, axes = plt.subplots(2, 1, figsize=(14, 8))
    rolling_sharpe = (returns.rolling(window).mean() / 
                      returns.rolling(window).std() * np.sqrt(252))
    axes[0].plot(rolling_sharpe.index, rolling_sharpe.values)
    axes[0].axhline(0, color='black', linestyle='--', alpha=0.5)
    axes[0].set_title(f'Rolling {window//21}M Sharpe Ratio')
    
    rolling_dd = returns.rolling(window).apply(
        lambda x: (x.cumsum().cummax() - x.cumsum()).max()
    )
    axes[1].plot(rolling_dd.index, rolling_dd.values)
    axes[1].set_title(f'Rolling {window//21}M Max Drawdown')
```

---

## 七、统计报送清单

策略开发者向用户汇报时，以下内容缺一不可：

| 类别 | 指标 | 必须 |
|------|------|------|
| 收益 | 年化收益率 | ✓ |
| 收益 | 累计收益率 | ✓ |
| 收益 | 超额收益（vs 基准） | ✓ |
| 收益分布 | Mean | ✓ |
| 收益分布 | Median | ✓ |
| 收益分布 | P5, P25, P40, P75, P95 | ✓ |
| 收益分布 | 偏度, 峰度 | ✓ |
| 风险 | 年化波动率 | ✓ |
| 风险 | 最大回撤 | ✓ |
| 风险 | VaR (95%, 99%) | 推荐 |
| 风险 | CVaR (95%, 99%) | 推荐 |
| 风险调整 | Sharpe ratio | ✓ |
| 风险调整 | NW 调整后 Sharpe | ✓ |
| 风险调整 | Sortino ratio | ✓ |
| 风险调整 | Calmar ratio | ✓ |
| 风险调整 | Information ratio | ✓ |
| 显著性 | OLS t-stat + p 值 | ✓ |
| 显著性 | NW t-stat + p 值 | ✓ |
| 显著性 | 显著性水平标注 | ✓ |
| 交易 | 胜率, 盈亏比 | ✓ |
| 交易 | 年化换手率 | ✓ |
| 交易 | 平均持仓天数 | 推荐 |
| 归因 | 因子暴露 + NW t | 可选 |
| 归因 | Brinson 归因 | 可选 |
