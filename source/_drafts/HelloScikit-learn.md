---
title: "HelloScikit-learn"
tags: [python, scikit]
categories: [Tech]
---


## Mac上安装Sci-learn套件
matplotlib是基于python的数据绘图框架，


用`pip`安装下面的包：

```
gnureadline==6.3.3
ipython==3.0.0
matplotlib==1.4.3
mock==1.0.1
nose==1.3.4
numpy==1.9.2
pandas==0.16.0
pyparsing==2.0.3
python-dateutil==2.4.2
pytz==2015.2
scikit-learn==0.16.0
scipy==0.15.1
six==1.9.0
sympy==0.7.6
```
在Mac上需要为matplotlib指定backend：

```bash
$ cat >> ~/.matplotlib/matplotlibrc << EOF
backend : TkAgg
EOF
```

简单测试：

```python
import numpy as np
import matplotlib.pyplot as plt

freqs = np.arange(2, 20, 3)
t = np.arange(0.0, 1.0, 0.001)
s = np.sin(2*np.pi*freqs[0]*t)
plt.plot(t, s, lw=2)
plt.show()
```

`ipython`很适合做前期的可视化分析。
`%pylab`

## Hello Pandas
Pandas准确说应该是矩阵计算框架。

```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt

s = pd.Series([1,3,5,np.nan,6,8])
dates = pd.date_range('20130101',periods=6)
df = pd.DataFrame(np.random.randn(6,4),index=dates,columns=list('ABCD'))
df.dtypes
df.head()
df.tail(3)
df.index
df.columns
df.values
df.describe()
df.T
df.sort_index(axis=1, ascending=False)
df.sort(columns='B')

# selecting data
df['A']
df[0:3]
df['20130102':'20130104']
df.loc[dates[0]]
df.loc[:,['A','B']]
df.loc['20130102':'20130104',['A','B']]
df.loc['20130102',['A','B']]
df.loc[dates[0],'A']
df.at[dates[0],'A'] # fast access

# select by position
df.iloc[3]
df.iloc[3:5,0:2]
df.iloc[[1,2,4],[0,2]]
df.iloc[1,1]

# select by condition
df[df.A > 0]
df[df > 0]
df > 0 # 会计算每个元素 > 0，返回bool矩阵

df2 = df.copy()
df2['E']=['one', 'one','two','three','four','three']
df2[df2['E'].isin(['two','four'])] # filtering

# setting
s1 = pd.Series([1,2,3,4,5,6],index=pd.date_range('20130102',periods=6))
df['F'] = s1 # 基于dates合并，F列第一个为'NaN'
df.at[dates[0],'A'] = 0
df.iat[0,1] = 0
df.loc[:,'D'] = np.array([5] * len(df))
df2[df2 > 0] = -df2

# missing data
df1 = df.reindex(index=dates[0:4],columns=list(df.columns) + ['E'])
df1.loc[dates[0]:dates[1],'E'] = 1
df1.dropna(how='any') # drop row where there's NaN
df1.fillna(value=5)
pd.isnull(df1) # find the NaN

# Operation
df.mean()
pd.Series([1,3,5,np.nan,6,8],index=dates)
s = pd.Series([1,3,5,np.nan,6,8],index=dates).shift(2)
df.sub(s,axis='index') # 每一列减去s对应的元素值

# Apply func to data
df.apply(np.cumsum) # 累积
df.apply(lambda x: x.max() - x.min())

# 离散
s = pd.Series(np.random.randint(0,7,size=10))
s.value_counts() # 索引value出现次数

# String
s = pd.Series(['A', 'B', 'C', 'Aaba', 'Baca', np.nan, 'CABA', 'dog', 'cat'])
s.str.lower()

# Merge
df = pd.DataFrame(np.random.randn(10, 4))
pieces = [df[:3], df[3:7], df[7:]]
pd.concat(pieces)

# Join
left = pd.DataFrame({'key': ['foo', 'foo'], 'lval': [1, 2]})
right = pd.DataFrame({'key': ['foo', 'foo'], 'rval': [4, 5]})
pd.merge(left, right, on='key')

# Append
df = pd.DataFrame(np.random.randn(8, 4), columns=['A','B','C','D'])
s = df.iloc[3]
df.append(s, ignore_index=True)

# Grouping
df = pd.DataFrame({'A' : ['foo', 'bar', 'foo', 'bar',
                             'foo', 'bar', 'foo', 'foo'],
                       'B' : ['one', 'one', 'two', 'three',
                             'two', 'two', 'one', 'three'],
                      'C' : np.random.randn(8),
                       'D' : np.random.randn(8)})
df.groupby('A').sum()
df.groupby(['A','B']).sum()

# Reshape
tuples = list(zip(*[['bar', 'bar', 'baz', 'baz',
   ....:                      'foo', 'foo', 'qux', 'qux'],
   ....:                     ['one', 'two', 'one', 'two',
   ....:                      'one', 'two', 'one', 'two']]))
index = pd.MultiIndex.from_tuples(tuples, names=['first', 'second'])
df2 = df[:4]
stacked = df2.stack()
stacked.unstack()
stacked.unstack(1)
stacked.unstack(0)

# Pivot Table
df = pd.DataFrame({'A' : ['one', 'one', 'two', 'three'] * 3,
   .....:                    'B' : ['A', 'B', 'C'] * 4,
   .....:                    'C' : ['foo', 'foo', 'foo', 'bar', 'bar', 'bar'] * 2,
   .....:                    'D' : np.random.randn(12),
   .....:                    'E' : np.random.randn(12)})

pd.pivot_table(df, values='D', index=['A', 'B'], columns=['C'])

# Time Series
rng = pd.date_range('1/1/2012', periods=100, freq='S')
ts = pd.Series(np.random.randint(0, 500, len(rng)), index=rng)
ts.resample('5Min', how='sum')

rng = pd.date_range('3/6/2012 00:00', periods=5, freq='D')
ts = pd.Series(np.random.randn(len(rng)), rng)
ts_utc = ts.tz_localize('UTC')
ts_utc.tz_convert('US/Eastern')

rng = pd.date_range('1/1/2012', periods=5, freq='M')
ts = pd.Series(np.random.randn(len(rng)), index=rng)
ps = ts.to_period()
ps.to_timestamp()

prng = pd.period_range('1990Q1', '2000Q4', freq='Q-NOV')
ts = pd.Series(np.random.randn(len(prng)), prng)
ts.index = (prng.asfreq('M', 'e') + 1).asfreq('H', 's') + 9
ts.head()

# Categoricals
df = pd.DataFrame({"id":[1,2,3,4,5,6], "raw_grade":['a', 'b', 'b', 'a', 'a', 'e']})
df["grade"] = df["raw_grade"].astype("category")
df["grade"]
df["grade"].cat.categories = ["very good", "good", "very bad"]
df["grade"] = df["grade"].cat.set_categories(["very bad", "bad", "medium", "good", "very good"])
df.sort("grade")
df.groupby("grade").size()

# Plotting
ts = pd.Series(np.random.randn(1000), index=pd.date_range('1/1/2000', periods=1000))
ts = ts.cumsum()
ts.plot()
plt.show()

df = pd.DataFrame(np.random.randn(1000, 4), index=ts.index, columns=['A', 'B', 'C', 'D'])
df = df.cumsum()
plt.figure(); df.plot(); plt.legend(loc='best')
plt.show()

# IO with csv
df.to_csv('foo.csv')
pd.read_csv('foo.csv')
```

## Pandas用来做stock分析

```python
import pandas.io.data as web
import datetime
import matplotlib.pyplot as plt

start = datetime.datetime(2015,4,20)
end = datetime.datetime(2015,4,24)
ticker = "AAPL"
f=web.DataReader(ticker,'yahoo',start,end)

plt.plot(f['Close'])
plt.title('AAPL Closing Prices')
plt.show()
```

```bash
$ brew install ta-lib # 安装TA-Lib
$ pip install Cython  # TA-Lib依赖
$ easy_install TA-Lib # 安装Python API
```


## 新浪API

历史：
http://biz.finance.sina.com.cn/stock/flash_hq/kline_data.php?&rand=random(10000)&symbol=sz000415&end_date=20090130&begin_date=19950802&type=plain

http://hq.sinajs.cn/list=sh600389

http://table.finance.yahoo.com/table.csv?s=600000.ss

http://blog.csdn.net/djun100/article/details/25748419


参考

1. [PythonVsRSeries][sci1]
2. [matplotlib official site][sci2]
3. [云中的数据科学: IPython 和 pandas 进行投资分析][sci3]
4. [10 Min to Pandas][sci4]


[sci1]:http://www.eickonomics.com/pages/PythonVsRSeries.html "PythonVsRSeries"
[sci2]:http://matplotlib.org/ "matplotlib official site"
[sci3]:http://www.ibm.com/developerworks/cn/cloud/library/cl-datascienceincloud/ "云中的数据科学: IPython 和 pandas 进行投资分析"
[sci4]:http://pandas.pydata.org/pandas-docs/stable/10min.html "10 Min to Pandas"