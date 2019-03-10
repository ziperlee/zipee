---
title: webserver异常-TooManyOpenFiles
date: 2019-03-10 17:32:36
tags:
- flask
- 问题分析
- linux
categories:
- 问题分析
---


### 问题现象：
{%note warning%}
- webserver服务不可用
- 查询日志得知异常为"... Too Many Open Files"
- 重启webserver后服务正常可用
{%endnote%}


<!--more-->
***
### 问题环境：
{%note info%}
- 使用nginx+gunicorn+flask搭建webserver的运行环境
- 环境进程文件句柄使用数上线未设置过为1024
- gunicorn max-connections=2048 workers=9
{%endnote%}
***

### 问题分析：

1. ulimit -n 设置过小
  - 问题原因比较好定位，google一下大概就确定了
  - 调大open files应该就可以了
2. gunicorn max-connections指单个workers还是gunicorn整体的最大连接数？
	- 找了官方文档和google的博客均没细说，如果是前者那是有文件句柄不够用的可能性，而后者则可能性不大，最后还是自己看了源码并通过siege进行性能测试得出结论，`gunicorn max-connections指单个workers`。

***

### 问题解决：
- 根据性能测试调大open files，并对数据数值添加监控。

***

### 问题总结：
- **linux open files**
```
	1. 查看系统最大可打开文件数
		cat /proc/sys/fs/file-max
	2. 查看进程最大可打开文件数
		cat /etc/security/limits.conf
	3. 查看指定进程文件打开数
		lsof -p pid | wc -l 
	4. 查看gunicorn fork出所有进程的文件打开数
		lsof -n|awk '{print $1}'|sort|uniq -c|sort -nr|grep gunicorn
	5. 进程 fork() 出来的子进程，会继承父进程的 limits 设定
```
- **gunicorn**
```
	1. work_connections指gunicorn单进程并发数
	2. 若gunicorn worker_connections < 当前并发数，请求至多会超时，而不会报连接文件句柄异常
	3、当worker_connections > 1 时 且程序可异步操作越久（sleep），gevent作用越明显
```
- **异常再次发生处理步骤**
```
	1. ps -ef |grep gunicorn -> pid
	2. cat /proc/pid/limits.conf
	3. lsof -p pid |wc -l
	4. sudo vi  /etc/security/limits.conf
		asset soft nofile 10240
		asset hard nofile 10240
	5. restart program && cat /proc/pid/limits.conf
```
- **性能测试**
	1. 性能测试主要使用了siege和locust，siege通过创建线程进行并发对测试机的影响较大，而locust通过协程并发请求，测试并发数能更高，该部分的具体内容会另起篇幅进行总结梳理。