---
title: "计算 A 股 beta"
description: 
date: 2025-11-12T10:07:41Z
image: 
math: 
license: 
hidden: false
comments: true
draft: false
categories:
    - 量化
tags:
    - beta估计
weight: 1 
---
## 概要

> 所有数据来源于[akshare](https://akshare.akfamily.xyz/data/),字段名做了修改并被保存在数据库中(duckdb)

这里介绍如何计算 CAPM 模型中的 beta

> 关于 beta 的介绍， 参考 Wikipedia[Beta (finance)](<https://en.wikipedia.org/wiki/Beta_(finance)>)

beta 描述的是一个股票的波动率相对于市场的波动率的一种度量, 所以需要同时用到个股的历史收益数据以及市场收益数据

> 这里的收益都是百分比变化; 另外， 我们使用上证指数的百分比变化作为市场收益数据

计算 beta 的核心是计算**个股超额收益 ~ 市场超额收益的**这个 OLS 的回归系数

### 数据准备

#### 个股收益数据

```python
stock_price  = conn.sql(f"""
                select
                    DISTINCT
                    t1.stock_code,
                    t1.date,
                    t1.close price,
                    t2.stock_name,
                    t2.industry
                fromstock_zh_a_hist_m t1 left join stock_individual_info_em t2 on t1.stock_code = t2.stock_code
                """)

def calculate_excess_returns(df, risk_free_rate=0):
    df =  df.sort_values(["stock_code", "date"])
    df["ret"] = df.groupby("stock_code")['price'].pct_change()
    df['ret'] = df.groupby('stock_code')['price'].pct_change()
    df['ret_excess'] = df['ret'] - risk_free_rate

    return df

# Apply the function to add ret_excess column
stock_price_df = calculate_excess_returns(stock_price.to_df())
```

#### 市场收益数据

上证指数

```python
  market_index_df = conn.sql(
    f"""
    SELECT DISTINCT date, close market_index FROM index_zh_sz_d ORDER BY date
    """
  ).to_df()


market_index_df["mkt_excess"] = market_index_df["market_index"].pct_change()
```

整合个股收益数据和市场收益数据

```python
stock_price_with_mkt_excess = stock_price_df.merge(market_index_df, left_on='date', right_on='date', how='left').dropna()
```

### 回归模型

对单一股票计算 beta(以万科000002为例)

```python
import statsmodels.formula.api as smf

model_fit = smf.ols(
    formula="ret_excess ~ mkt_excess", data=stock_price_with_mkt_excess .query("stock_code == '000002'")
).fit()
coefficients = model_fit.summary2().tables[1]
coefficients
```

输出结果如下：

|            | Coef.     | Std.Err. | t         | P>\|t\|      | [0.025    | 0.975]   |
| ---------- | --------- | -------- | --------- | ------------ | --------- | -------- |
| Intercept  | -0.003801 | 0.006818 | -0.557436 | 5.779298e-01 | -0.017255 | 0.009654 |
| mkt_excess | 1.062607  | 0.127523 | 8.332703  | 2.068179e-14 | 0.810957  | 1.314258 |

可以看到万科的收益和市场收益（2010 年到 2025 年间的数据）是存在明显的相关性的,且 beta>1

> 这里的 Intercept 是截距，就是我们通常说的 alpha， 不过这里它并不显著

### 滚动窗口

上面我们使用了所有历史数据计算 beta，但是更加适合的方式是使用滚动窗口，比如我只用最近两年的数据来计算 beta（这样能更好地反映和捕捉股票本身对市场反应的变化）

```python
def estimate_capm(data, min_obs=1):
    if data.shape[0] < min_obs:
        capm = pd.DataFrame()
    else:
        fit = smf.ols(formula="ret_excess ~ mkt_excess", data=data).fit()
        coefficients = fit.summary2().tables[1]

        capm = pd.DataFrame(
            {
                "coefficient": coefficients.index,
                "estimate": coefficients["Coef."],
                "t_statistic": coefficients["t"],
            }
        ).assign(
            coefficient=lambda x: np.where(
                x["coefficient"] == "Intercept", "alpha", x["coefficient"]
            )
        )

    return capm

def roll_capm_estimation(data, look_back=60, min_obs=48):
    results = []
    dates = data["date"].sort_values().drop_duplicates()

    for i in range(look_back - 1, len(dates)):
        start_date = dates.iloc[i - look_back + 1]
        end_date = dates.iloc[i]

        window_data = data.query("date >= @start_date & date <= @end_date")

        result = estimate_capm(window_data)
        result["date"] = np.max(window_data["date"])
        results.append(result)

    if results:
        rolling_capm_estimation = pd.concat(results, ignore_index=True)
    else:
        rolling_capm_estimation = pd.DataFrame()

    return rolling_capm_estimation
```

**estimate_capm**会为每一个时间窗口计算 CAPM 的估计值, **roll_capm_estimation**则会批量计算多个时间窗口的 CAPM 估计值

```python
WINDOW_SIZE = 60
capm = (
    stock_price_with_mkt_excess.groupby("stock_code"）
    .apply(lambda x: roll_capm_estimation(x, WINDOW_SIZE), include_groups=False)
    .reset_index()
    .get(["stock_code", "date", "coefficient", "estimate", "t_statistic"])
)
```

### 并行计算

上面的方案的问题在于速度太慢，我们可以使用并行计算来提高速度。
#### 方案1：使用joblib
```python
from joblib import Parallel, delayed, cpu_count
# 全量股票数据
nested_data = stock_price_with_mkt_excess.groupby("stock_code", group_keys=True)
n_cores = cpu_count() - 1

capm = pd.concat(
    Parallel(n_jobs=n_cores)(
        delayed(lambda name, group: roll_capm_estimation(group).assign(stock_code=name))(
            name, group
        )
        for name, group in nested_data
    )
).get(["stock_code", "date", "coefficient", "estimate", "t_statistic"])
```
#### 方案2： 使用ProcessPoolExecutor
除了使用joblib，还可以使用python自带的ProcessPoolExecutor（推荐，更加pythonic）
```python
from concurrent.futures import ProcessPoolExecutor
import pandas as pd

def process_group(name, group):
    result = roll_capm_estimation(group)
    result['stock_code'] = name
    return result

# 使用 ProcessPoolExecutor 实现并行计算
with ProcessPoolExecutor(max_workers=n_cores) as executor:
    futures = [executor.submit(process_group, name, group) for name, group in nested_data]
    results = [future.result() for future in futures]

capm = pd.concat(results).get(["stock_code", "date", "coefficient", "estimate", "t_statistic"])
```

#### 方案3：使用duckdb
略

#### 保存 beta 数据

```python
capm.query("coefficient == 'mkt_excess'").rename(columns={"estimate": "beta"}).get(["stock_code", "date", "beta"]).to_csv(f"capm_{WINDOW_SIZE}.csv", index=False)
```

### 可视化分析[可选]

```python
from lets_plot import *

plot = ggplot(capm, aes(x="date", y="estimate", color="stock_name")) +\
    geom_line(size=1.5) +\
    labs(
        x="",
        y="",
        color="",
        linetype="",
        title="股票月度贝塔系数估计值(5年滚动窗口数据)",
    ) +\
    ggsize(800, 600) +\
    theme(
        legend_position="right",
        plot_margin=[10, 30, 0, 0]  # Add right margin: [top, right, bottom, left]
    )

plot
```

### 参考

[beta-estimation](https://www.tidy-finance.org/python/beta-estimation.html)
