---
title: "双因子排序及因子构建"
description: 
date: 2025-11-14T02:16:24Z
image: 
math: 
license: 
hidden: false
comments: true
draft: false
categories:
    - 量化
tags:
    - 因子投资
weight: 1 
---
### 数据准备
#### 股价数据

```python
# stock price
conn = duckdb.connect(DB_FILE, read_only=True)

stock_price_table_name = "stock_zh_a_hist_m" if MODE == "monthly" else "stock_zh_a_hist_d"

stock_price  = conn.sql(f"""
                select
                    DISTINCT
                    t1.stock_code,
                    t1.date,
                    t1.close price,
                    t2.stock_name,
                    t2.mktcap
                from {stock_price_table_name} t1
                left join stock_individual_info_em t2 on t1.stock_code = t2.stock_code
                """)
stock_price_df = calculate_excess_returns(stock_price.to_df()).dropna()
```

> 警告：这里的数据其实存在一个非常明显的问题，市值规模数据mktcap正常情况下应该使用历史数据， 这里用的是当前数据， 所以评估结果本身不具备参考价值

#### 去除小盘股数据

```python
DROP_RATE = 0.30
small_caps = stock_price_df.sort_values(by=["stock_code", "date"]).drop_duplicates(subset=["stock_code"], keep="last").sort_values(by="mktcap", ascending=True)

small_cap_stocks = small_caps.head(int(len(small_caps) * DROP_RATE)).stock_code

stock_price_df = stock_price_df.sort_values(by=["stock_code", "date"]).query("stock_code not in @small_cap_stocks")
```

#### be 数据
be也就是book-equity
```python
be = conn.sql(f"""
              SELECT
                DISTINCT
                stock_code,
                be,
                date
              FROM stock_zcfz_em
              """
)

book_equity = be.to_df().query("stock_code in @stock_price_df['stock_code'].unique()")
```

#### 市值

将市值数据滞后一周,防止 look-ahead bias(我们只能用较早的财务数据 而不能用最新的 因为最新的往往在当时还未取得)

```
size = (stock_price_df
  .assign(sorting_date=lambda x: x["date"]+pd.DateOffset(months=1))
  .rename(columns={"mktcap": "size"})
  .get(["stock_code", "sorting_date", "size"])
)
```

然后计算 bm

bm = book_equity / market_cap ,理论上越大公司就越有价值

```python
bm = (book_equity
  .merge(stock_price_df, how="inner", on=["stock_code", "date"])
  .assign(bm=lambda x: x["be"]/x["mktcap"],
          sorting_date=lambda x: x["date"]+pd.DateOffset(months=6))
  .assign(accounting_date=lambda x: x["sorting_date"])
  .get(["stock_code", "sorting_date", "accounting_date", "bm"])
)
```

同样， 我们对数据进行滞后处理（这里用了半年前的数据， 同时限定财务数据必须在一年以内, 不然没有参考价值）

#### 构建完整数据

```python
data_for_sorts = (stock_price_df
  .merge(bm,
         how="left",
         left_on=["stock_code", "date"],
         right_on=["stock_code", "sorting_date"])
  .merge(size,
         how="left",
         left_on=["stock_code", "date"],
         right_on=["stock_code", "sorting_date"])
  .get(["stock_code", "date", "ret_excess",
        "mktcap", "size", "bm", "accounting_date"])
)


data_for_sorts = (data_for_sorts
  .sort_values(by=["stock_code","date"])
  .groupby(["stock_code", ])
  .apply(lambda x: x.assign(
      bm=x["bm"].fillna(method="ffill"),
      accounting_date=x["accounting_date"].fillna(method="ffill")
    )
  )
  .reset_index(drop=True)
  .assign(threshold_date = lambda x: (x["date"]-pd.DateOffset(months=12)))
  .query("accounting_date > threshold_date")   # 保证财务数据是一年内的
  .drop(columns=["accounting_date", "threshold_date"])
  .dropna()
)
```

### 计算 SMB 因子和 HML 因子

SMB = small - big, 表示小市值公司和大公司市值的收益差值
HML = high - low, 表示高价值(高 bm)和低 bm 的收益差值

```python
portfolios['portfolio_size'] = portfolios['portfolio_size'].astype(int)
portfolios['portfolio_bm'] = portfolios['portfolio_bm'].astype(int)

n_size = portfolios['portfolio_size'].max()
n_bm = portfolios['portfolio_bm'].max()

# SMB（小盘 - 大盘），逐日计算，并处理缺失（若某日某一侧缺失则结果为 NaN）
small_ret = portfolios.loc[portfolios['portfolio_size'] == 1].groupby('date')['ret'].mean()
big_ret   = portfolios.loc[portfolios['portfolio_size'] == n_size].groupby('date')['ret'].mean()
smb = (small_ret - big_ret).rename('SMB').reset_index()

# HML（高 BE/ME - 低 BE/ME）
high_ret = portfolios.loc[portfolios['portfolio_bm'] == n_bm].groupby('date')['ret'].mean()
low_ret  = portfolios.loc[portfolios['portfolio_bm'] == 1].groupby('date')['ret'].mean()
hml = (high_ret - low_ret).rename('HML').reset_index()

# 若希望将 SMB/HML 合并成一个 DataFrame
factors = smb.merge(hml, on='date', how='outer')
```

这里我们使用独立排序(注意fama-french 3因子模型使用的是 double sort!)， 分别计算了 SMB 和 HML

下面是这两个因子均值的汇总:
SMB： -0.004243
HML： -0.017698

然后分别使用截距项回归验证这两个因子的作用

```python
import statsmodels.api as sm
from regtabletotext import prettify_result


model_fit_smb = (sm.OLS.from_formula(
    formula="SMB ~ 1",
    data=factors
  )
  .fit(cov_type="HAC", cov_kwds={"maxlags": 6})
)
prettify_result(model_fit_smb)
```

```
model_fit_hml = (sm.OLS.from_formula(
    formula="HML ~ 1",
    data=factors
  )
  .fit(cov_type="HAC", cov_kwds={"maxlags": 6})
)
prettify_result(model_fit_hml)
```

也就是说， 至少在 A 股, SMB 以及 HML 并不是显著有效的因子

### 讨论
在石川的<<因子投资>>中论证过 SMB 和 HML 这两个重要因子，在书中这两个因子的效用还是得到了验证;   
以规模效应（SMB）为例， 在石川的书中，对小盘股进行了筛选（由于 A 股特有的借壳上市机制, 小盘股）剔除了市值最小的 30%的股票， 另外他的时间跨度是 2000-2020 年. 书中也提到了再 2015 年之前规模因子是有效的， 但是之后SMB效果就不再明显；  
还有一点， 前面我也提到， 由于历史市值规模的缺失，导致我的评估结果没有办法真实反映因子的作用。


### 参考
[value-and-bivariate-sorts](https://www.tidy-finance.org/python/value-and-bivariate-sorts.html)  
因子投资, 石川