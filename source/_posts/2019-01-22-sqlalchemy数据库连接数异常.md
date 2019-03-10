---
title: sqlalchemy数据库连接数异常
date: 2019-01-22 21:03:19
tags:
- mysql
- 问题分析
categories:
- 问题分析
---

### 问题现象：
{%note warning%}
- web端调用flask接口阻塞，最终超时返回系统异常
- 查询日志得知具体异常为超出数据库最大连接数
- 重新flask webserver后数据库连接数正常释放
- 持续使用系统一段时间后数据库连接数再次封顶
- 环境操作用户寥寥无几
{%endnote%}


<!--more-->
***
### 问题环境：
{%note info%}
- 使用flask搭建webserver的运行环境
- 使用flask-sqlalchemy连接数据库并进行相关业务操作，SQLALCHEMY_POOL_SIZE = 64， SQLALCHEMY_POOL_RECYCLE = 30
- mysql设置最大连接数200
- 宿主机CPU 2核心
- 通过进程数2*2+1 = 5
{%endnote%}
***

### 问题分析：

1. 怀疑对flask-sqlalchemy使用不当导致
  - 问题原因明显是数据库连接使用后没有释放
  - 数据库操作业务层均使用封装的方法，不存在操作对数据库的连接和释放
  - flask-sqlalchemy的连接释放由`teardown_appcontext`钩子统一处理
  ```python
	@app.teardown_appcontext
	def shutdown_session(response_or_exc):
	    if app.config['SQLALCHEMY_COMMIT_ON_TEARDOWN']:
	        if response_or_exc is None:
	            self.session.commit()
	
	    self.session.remove()
	    return response_or_exc
	```
2. mysql数据库连接超出首因
	- (2\*2+1) * 64 > 200 
	- 困惑连接数回收已经指定`SQLALCHEMY_POOL_RECYCLE=30`但未生效
	- 对`SQLALCHEMY_POOL_RECYCLE`理解错误，该项真实含义为queue_pool可用连接的回收时间
	- 当前问题为连接未释放，顾与`SQLALCHEMY_POOL_RECYCLE`无关
3. 若(2\*2+1) * 64 < 200 会不会好点？
	- 异常改为`QueuePool limit of size <x> overflow <y> reached, connection timed out, timeout <z>
`
	- 达到queue_pool的最大限制后同样是无法连接数据库，只是换了种“死法”
	
**分析到这里仍旧没有头绪...**

***

### 问题排查：
- **查询数据库连接数** 
  ```shell
	SELECT * FROM INFORMATION_SCHEMA. PROCESSLIST
	```
- **linux下查询连接端口对应的进程**
	```shell
	lsof -i:48057|grep celery|awk   '{print$2}'|xargs -I{} echo 'ps -ef|grep {}'|bash
	```
- **windows下查询连接端口对应的进程**
  ```shell
	netstat -ano|findstr "8080"
	tasklist|findstr [进程号]
	```
***

### 问题解决：
- 问题调试发现单接口调用连接数并未按预期增长
- 通过页面使用操作连接数会不规则的增长
- 逐个接口比对，最终发现在调用sse长连接接口后，连接数会稳定增长1
- 分析sse仅操作了redis，并未存在对mysql的相关操作
- 调试发现调用sse会进入@after_request请求钩子中，而在该钩子处理函数中使用了current_user，即调用了数据库
- 到此真相大白，代码修改很简单过滤该url既可，但问题的思考排查真实饶了一大圈
***

### 问题总结：
- 最后反思，其实在确定flask-sqlalchemy框架层在正常api逻辑处理中会自动释放连接后，就应该直接猜测是非业务短连接接口导致，但这其中的盲点主要在sse正常情况下是与mysql毫不相干的，顾很容易忽略sse的长连接问题
***
