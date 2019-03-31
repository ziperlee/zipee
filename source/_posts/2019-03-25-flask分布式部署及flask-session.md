---
title: flask分布式部署及flask-session
date: 2019-03-25 21:12:52
tags:
- flask 
- session
- 分布式
categories:
- flask
---

{%note info%}
随着业务系统访问量的增长，多次部署成了必然，下面来聊聊flask的分布式部署以及原理。
{%endnote%}
<!--more-->

### flask默认的session做了什么
flask作为web应用框架若多机部署，第一个问题是需要一个请求接入网关，通常我们使用nginx统一进行流量的分发。
但随之而来会有一个新的问题，即flask的session多机之间会共享吗？带着这个问题，我们看看flask关于session的源码：
```python
class SecureCookieSession(CallbackDict, SessionMixin):
	...
class SecureCookieSessionInterface(SessionInterface):
	def open_session(self, app, request):
		...
	def save_session(self, app, session, response):
		...
```
以上是flask.sessions.py实现的主要框架：
- SecureCookieSession即flask的session类，可以简单的理解成一个dict对象。
- SecureCookieSessionInterface即flask的session接口类，open_session方法用于创建session，save_session方法用于将session加密并存放在response的cookie中。所以flask是默认将用户的session存储在客户端的cookie中，这样请求-应答的数据中就有了用户操作的上下文了，至于这么做的优劣将在下文分析。

### 常见的分布式部署session解决方案

- 服务器间session复制
session复制是早期的企业级的使用比较多的一种服务器集群session管理机制。应用服务器开启web容器的session复制功能，在集群中的几台  服务器之间同步session对象，使得每台服务器上都保存所有的session信息，这样任何一台宕机都不会导致session的数据丢失，服务器使用session时，直接从本地获取。
像java的一些应用服务器，如tomcat等自带次功能。在python-web不常见，`缺点`是session同步会暂用内网网络带宽，且服务器水平扩展存在明显上线。
- session与服务器绑定
通过请求网关，如nginx，将负载均衡的策略改成ip-hash的模式，即用户的每次请求都会分发到同一台服务器，那么sesison则能够正常的被解析。
`优点`：无需修改业务代码
`缺点`：缺乏高可用性，当其中一台服务器宕机，该机器上用户需要重新登录到其他服务器
- 客户端session存储
即flask默认的session存储方案，可见什么都不需要改动，flask已经支持水平扩展，细心的童鞋想想当flask通过gunicorn启动时多进程为啥能够共享session，即不难想到多个服务间共享应该也问题不大。这里注意的是多服务器间的secret_key必须相同。
`优点`：无需改造，flask默认支持
`缺点`：
	- session数据存储在客户端，即使加密也还是一件存在泄露风险的事情
	- session数据占用外网带宽
	- 受cookie的大小限制，session能记录的数据有限
- 服务端session统一存储
对session进行统一的存储，所有服务器共享该存储服务上的数据
`优点`：服务水平扩展性良好，服务端存储，安全
`缺点`：
	- 每次请求至少需要一次内部网络请求，占用网络带宽
	- 需要侵入业务代码

### flask-session服务端session存储
通过比较不难发现，服务端session统一存储是最合适的解决方案。
那么我们来谈谈怎么实现，幸运的是已经有前任实现了flask对应的扩展包`flask-session`，我们一起看看它的实现：
代码大概500+行，但我们实际用到的可能就几十行。
- 首先我们需要选择session寄存的服务，flask-session支持`redis`,`memcached`,`filesystem`,`mongodb`,`sqlalchemy`作为存储介质
- 以redis举例，再看代码：
```python
class RedisSessionInterface(SessionInterface):
    serializer = pickle
    session_class = RedisSession

    def __init__(self, redis, key_prefix, use_signer=False, permanent=True):
        if redis is None:
            from redis import Redis
            redis = Redis()
        self.redis = redis
        self.key_prefix = key_prefix
        self.use_signer = use_signer
        self.permanent = permanent
	def open_session(self, app, request):
		...
	def save_session(self, app, session, response):
		...
```
重写`open_session，save_session`，将session（dict）存储在redis并将session_id（key）返回给客户端


### flask http请求-应答完整的数据流
客户端http请求 
-> 服务端负载均衡至随机服务器 
-> 应用上下文入栈(app_ctx) 
-> 请求上下文入栈(request_ctx),同时生成session 
-> 通过request_ctx中的路由信息找到视图函数(view_func) 
-> view_func进行业务处理 
-> 应用上下文出栈(app_ctx)
-> 请求上下文出栈(request_ctx) 
-> 保存session或sessino_id进cookie 
-> 返回应答 
-> 数据写入对应的文件描述符并刷新
其实flask的源码阅读起来并不吃力，看下来会发现flask框架代码的思路结构非常的清晰，并惊叹于这个框架的可扩展性，flask的源码非常值得学习和借鉴。