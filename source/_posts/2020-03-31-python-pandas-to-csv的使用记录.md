---
title: python-pandas.to_csv的使用记录
date: 2020-03-31 19:46:42
tags:
- pandas
categories:
- python
---
{%note info%}
记录下pandas.to_csv函数使用的坑。
{%endnote%}
<!--more-->

- 本想聊聊事情起因，经过，结果，奈何手懒，直接记录下结果吧
- 问题：
    - 1、pandas读取csv，再to_string，再转回pandas.dataframe，再保存为csv
    - 2、pandas读取csv，再保存为csv
    - 1，2保存的数据结果不一致
- 样本数据：

```
# sample.csv
a,b,c,d,e,f
11.0,22.0,1.0,2.0,3.0,4.0
22.0,33.0,1.0,2.0,3.0,4.0
33.0,33.0,1.0,2.0,3.0,4.0
44.0,33.0,1.0,2.0,3.0,4.0
55.0,33.0,1.0,2.0,3.0,4.0
66.0,33.0,1.0,2.0,3.0,4.0
77.0,33.0,1.0,2.0,3.0,4.0
88.0,33.0,1.0,2.0,3.0,4.0
```
- 涉及的代码也简单，如下：

```python
from io import StringIO

import pandas as pd


# 1
pdf = pd.read_csv('sample.csv', sep='\s+')
pdf.to_csv('/tmp/sample2.csv', index=0, float_format='%.10g')

# 2
pdf2 = pd.read_csv(StringIO(pdf.to_string()), sep='\s+')
pdf2.to_csv('/tmp/sample2.csv', index=0, float_format='%.10g')

```

- 1的保存结果：

```
11.0,22.0,1.0,2.0,3.0,4.0
22.0,33.0,1.0,2.0,3.0,4.0
33.0,33.0,1.0,2.0,3.0,4.0
44.0,33.0,1.0,2.0,3.0,4.0
55.0,33.0,1.0,2.0,3.0,4.0
66.0,33.0,1.0,2.0,3.0,4.0
77.0,33.0,1.0,2.0,3.0,4.0
88.0,33.0,1.0,2.0,3.0,4.0
```

- 2的保存结果：

```
11,22,1,2,3,4
22,33,1,2,3,4
33,33,1,2,3,4
44,33,1,2,3,4
55,33,1,2,3,4
66,33,1,2,3,4
77,33,1,2,3,4
88,33,1,2,3,4
```

- 解决方案：
```python
# 去除float_format='%.10g'
pdf2.to_csv('/tmp/sample2.csv', index=0)
```