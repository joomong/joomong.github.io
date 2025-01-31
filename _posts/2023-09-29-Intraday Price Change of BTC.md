---
title: Intraday Price Change of BTC
author: Joomong
date: 2023-09-13 11:33:00 +0900
categories: [Blogging, HFT]
tags: [HFT]
pin: false
math: true
sitemap:
  changefreq: daily
  priority: 1.0
# image:
#   path: /commons/devices-mockup.png
#   lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
#   alt: Responsive rendering of Chirpy theme on multiple devices.
# mermaid: true

---


```python
import numpy as np
import pandas as pd
import seaborn as sns
import matplotlib.pyplot as plt
```


```python
header = ['trade_id','price','quantity','order_id','timestamp','is_buyer_maker']
df = pd.read_csv('./BTCUSDT-trades-2023-05-31.csv',header=0 , names = header)
df.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>trade_id</th>
      <th>price</th>
      <th>quantity</th>
      <th>order_id</th>
      <th>timestamp</th>
      <th>is_buyer_maker</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>3765419375</td>
      <td>27680.0</td>
      <td>0.400</td>
      <td>11072.0000</td>
      <td>1685491200110</td>
      <td>True</td>
    </tr>
    <tr>
      <th>1</th>
      <td>3765419376</td>
      <td>27680.0</td>
      <td>0.006</td>
      <td>166.0800</td>
      <td>1685491200156</td>
      <td>True</td>
    </tr>
    <tr>
      <th>2</th>
      <td>3765419377</td>
      <td>27680.0</td>
      <td>0.300</td>
      <td>8304.0000</td>
      <td>1685491200157</td>
      <td>True</td>
    </tr>
    <tr>
      <th>3</th>
      <td>3765419378</td>
      <td>27680.1</td>
      <td>0.003</td>
      <td>83.0403</td>
      <td>1685491200244</td>
      <td>False</td>
    </tr>
    <tr>
      <th>4</th>
      <td>3765419379</td>
      <td>27680.1</td>
      <td>0.003</td>
      <td>83.0403</td>
      <td>1685491203809</td>
      <td>False</td>
    </tr>
  </tbody>
</table>
</div>



# Introduction

In many High-Frequency Trading papers, it is common to make the following assumptions:

- The time duration $\Delta t$ is sufficiently small.
- The terminal time $T$ is also small.  

Therefore, it is necessary to investigate the characteristics of high-frequency trading data.


```python
def closed_time_series(df, time_interval):
    """
    Change tick data to OHLCV data
    """
    df['timestamp'] = pd.to_datetime(df['timestamp'], unit='ms')
    df = df.set_index('timestamp')
    df['quantity'] = df['quantity'] * df['price']
    df['buy_quantity'] = np.where(df['is_buyer_maker'], df['quantity'], 0)
    df['sell_quantity'] = np.where(df['is_buyer_maker'], 0, df['quantity'])
    df = df.resample(str(time_interval)+'S').agg({'price':'last', 'quantity':'sum', 'buy_quantity':'sum', 'sell_quantity':'sum'})
    df['return'] = df['price'].diff()
    df['log_return'] = np.log(df['price']) - np.log(df['price'].shift(1))
    df = df.dropna()
    df['volatility'] = df['return'].rolling(600).std()
    return df
```

According to Tsay's "Financial Time Series," there are interesting characteristics of intraday price changes, including a concentration on "no change" and discreteness. Let's investigate these characteristics step by step.

## Concentration on "no change"
To examine the concentration of price changes around zero, we can calculate the frequency distribution of price changes at small time intervals (e.g., seconds or minutes).   
By analyzing the distribution, we can determine if there is a significant number of instances where the price remains unchanged or experiences minimal fluctuations.


```python
one_sec = closed_time_series(df, 1)[:600]
one_min = closed_time_series(df, 60)[:600]

fig, axs = plt.subplots(1, 2, figsize=(15, 5))
fig.suptitle('1sec vs 60sec')
sns.histplot(data=one_sec, x='return', bins=100, ax=axs[0], kde=True)
sns.histplot(data=one_min, x='return', bins=100, ax=axs[1], kde=True)
axs[0].set_ylabel('frequency')
axs[1].set_ylabel('frequency')
plt.show()
```


    
![png](/assets/img/HF-data_files/HF-data_7_0.png)
    


## Discreteness
The discreteness refers to the occurrence of discrete jumps or sudden changes in prices rather than continuous movements. This can be observed by plotting the intraday price series and examining whether there are frequent instances where prices exhibit sharp discontinuities or large jumps within short time intervals.


```python
fig, axs = plt.subplots(2, 1, figsize=(15, 5))
# one_min_slice = one_min[:600]
# one_sec_slice = one_sec[:600]
fig.suptitle('1sec vs 60sec')
sns.lineplot(data=one_sec, x=one_sec.index, y='price', ax=axs[0])
sns.lineplot(data=one_min, x=one_min.index, y='price', ax=axs[1])
plt.show()
```


    
![png](/assets/img/HF-data_files/HF-data_9_0.png)
    


## Exception Case : BTC/TUSD with zero trading fee
Recently, Binance initiated a zero trading fee promotion for the BTC/TUSD spot pair. This is an excellent opportunity for both regular traders and market makers as they can trade without incurring any fees. However, this has led to some interesting phenomena. In very short time durations, there are numerous price movements which are continuous.

This might be due to the increase in 'noise traders' who operate on incomplete information and typically have shorter trading durations. The absence of trading fees may encourage more noise traders to participate, leading to more frequent and continuous price movements.

However, a comprehensive understanding of these phenomena would require expertise in market microstructure theory.


```python
header = ['number','price','quantity','start trade','end trade','timestamp','s','is_buyer_maker']
df = pd.read_csv('./BTCTUSD-aggTrades-2023-06-27.csv',header=0 , names = header)
df.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>number</th>
      <th>price</th>
      <th>quantity</th>
      <th>start trade</th>
      <th>end trade</th>
      <th>timestamp</th>
      <th>s</th>
      <th>is_buyer_maker</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>173017437</td>
      <td>30295.75</td>
      <td>0.00181</td>
      <td>215415073</td>
      <td>215415073</td>
      <td>1687824000301</td>
      <td>True</td>
      <td>True</td>
    </tr>
    <tr>
      <th>1</th>
      <td>173017438</td>
      <td>30295.75</td>
      <td>0.00472</td>
      <td>215415074</td>
      <td>215415074</td>
      <td>1687824000308</td>
      <td>True</td>
      <td>True</td>
    </tr>
    <tr>
      <th>2</th>
      <td>173017439</td>
      <td>30295.72</td>
      <td>0.00128</td>
      <td>215415075</td>
      <td>215415075</td>
      <td>1687824000308</td>
      <td>True</td>
      <td>True</td>
    </tr>
    <tr>
      <th>3</th>
      <td>173017440</td>
      <td>30295.64</td>
      <td>0.00132</td>
      <td>215415076</td>
      <td>215415076</td>
      <td>1687824000308</td>
      <td>True</td>
      <td>True</td>
    </tr>
    <tr>
      <th>4</th>
      <td>173017441</td>
      <td>30295.60</td>
      <td>0.00400</td>
      <td>215415077</td>
      <td>215415078</td>
      <td>1687824000308</td>
      <td>True</td>
      <td>True</td>
    </tr>
  </tbody>
</table>
</div>




```python
one_sec = closed_time_series(df, 1)[:600]
one_min = closed_time_series(df, 60)[:600]

fig, axs = plt.subplots(1, 2, figsize=(15, 5))
fig.suptitle('1sec vs 60sec')
sns.histplot(data=one_sec, x='return', bins=100, ax=axs[0], kde=True)
sns.histplot(data=one_min, x='return', bins=100, ax=axs[1], kde=True)
axs[0].set_ylabel('frequency')
axs[1].set_ylabel('frequency')
plt.show()
```


    
![png](/assets/img/HF-data_files/HF-data_12_0.png)
    



```python
fig, axs = plt.subplots(2, 1, figsize=(15, 5))
# one_min_slice = one_min[:600]
# one_sec_slice = one_sec[:600]
fig.suptitle('1sec vs 60sec')
sns.lineplot(data=one_sec, x=one_sec.index, y='price', ax=axs[0])
sns.lineplot(data=one_min, x=one_min.index, y='price', ax=axs[1])
plt.show()
```


    
![png](/assets/img/HF-data_files/HF-data_13_0.png)
    

