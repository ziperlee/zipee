---
title: 关于pyspark-dataframe与pandas-dataframe的那些事
date: 2020-03-27 21:31:17
tags:
- pysqark
- pandas
categories:
- python
---
{%note info%}
pyspark.dataframe与pandas.dataframe大概目前机器学习算子中最长用到的数据结构了，本文来讲讲各自的常用操作和两者间相互转换。
{%endnote%}
<!--more-->

## pyspark
- 创建spark-session

```python
from pyspark.sql import SparkSession
session = (
    SparkSession.builder.appName("spark_pyspark")
        .config("hive.metastore.uris", "thrift://ip:9083")
        .config("spark.sql.warehouse.dir", "/user/hive/warehouse")
        .enableHiveSupport()
        .getOrCreate()
)
```

- 通过pyspark-sql进行数据查询、聚合、统计

```python
df = session.sql("select * from table_name")
```

- dataframe数据落地保存

```python
// 类似的常用的方法还有saveAsPickleFile
df.rdd.saveAsTextFile('/tmp/text_file')

// 通过spark-sql
df.write.saveAsTable(output_table, None, "overwrite", None)
```

- 重新读取落地的文件
	   
```python
// p类型为list
p = session.sparkContext.pickleFile('/tmp/pickle_file/part-00000').collect()
```

- 重新回到dataframe

```python
df = session.createDataFrame(p)
```

- datafram实例化

```python
// csv to dataframe
// schema数据类型为 `from pyspark.sql.types import *`
df = (
    self.session.read.format("com.databricks.spark.csv")
    .options(
        header=True,
        schema=schema,
        encoding=encoding,
        delimiter=col_delimiter,
    )
    .load(uri) // uri 可以指定hdfs文件或者本地文件
)

// mysql to dataframe
df = (
    self.session.read.format("jdbc")
    .options(
        url=url,
        driver="com.mysql.jdbc.Driver",
        dbtable=db_table_name,
        user=user_name,
        password=password,
    )
    .load()
)
```

- pyspark.dataframe to pandas.dataframe

```python
pdf = df.toPandas()
```

## pandas
- pandas.dataframe 实例化

```python
import pandas as pd

// csv to pdf
pdf = pd.read_csv(file_path)
```

- dataframe to csv

```python
pdf.to_csv(file_path, index=False)
```

- pandas.dataframe to pyspark.dataframe

```python
// session的声明方式见上文
df = session.createDataFrame(pdf)
```

- pdf to string

```python
s = pdf.to_string()
```

## 复杂问题

### spark.dataframe如何实现多机同步

正常情况下通过`df.write`落hdfs(上层可能基于hive或者spark-sql)实现共享。
但若要跨集群共享就要另想办法了。
解决方案就是将落地数据单独传输：

1、df -> hdfs -> local file -> internet
2、df -> pdf -> byte or string -> internet
显然第二种方法只需要落地一次，更优

```python
# spark df to pandas df
pdf = df.toPandas()

# pandas df to string
s = pdf.to_string()

# string to bytes
b = str.encode(s)

# send

# bytes to pandas df
from io import StringIO
s = str(bytes_data,'utf-8')
// sep 必须要指定，非则将结构混乱
df = pd.read_csv(StringIO(s), sep="\s+")
```

### pandas.dataframe数据精度问题
当pandas.dataframe to_csv或者转为pysprk.dataframe时，会默认进行精度转换，
to_csv中提供了指定精度的参数，然而每列特征的精度本就可能不一致，所以最好还是完整保留当前的数据内容。
解决方案：
1、通过float_format指定最长有效数字，`g`不补零，`f`补零

```python
pdf.to_csv(file_path, index=False, float_format="%.10g")
```
2、使用`pdf.to_string`的方式，保存数据快照，然后手动进行数据转换为csv。