---
title: flask-caching源码浅析
date: 2019-07-14 19:49:26
tags:
- flask
- python
- 源码浅析
categories:
---

{%note info%}
flask-caching源码浅析。
{%endnote%}
<!--more-->

## 简介
- flash-caching是一款flask开发技术栈中使用频率非常高的一款插件，起初的名称为“flask-cache”，由于原作者不再维护，后由其他人在其基础上开发维护了flask-caching。
- flask-caching支持后端存储类型（本文主要围绕使用redis的源码展开）：
	- simple 程序内部缓存
	- filesystem 系统文件缓存
	- redis
	- redissentinel
	- uwsgi
	- memcached
	- gaememcached
	- saslmemcached
	- spreadsaslmemcached

## 简单使用
```python
from flask_caching import Cache
cache = Cache()

config = {"CACHE_TYPE": "redis", "CACHE_REDIS_URL": app.config["CACHE_REDIS_URL"]}
    cache.init_app(app, config=config)

@app.route('/cache_test/<int:id>')
@cache.cached(timeout=60 * 5)
def cache_test(id):
    from flask import request
    got_data = request.get_json()
    import time

    return f"{got_data}.{id}.{time.time()}"

@cache.memoize(timeout=60 * 5)
def f_cache(a):
    return a

@app.route('/cache_test2/<int:id>')
def cache_test2(id):
    return f"{f_cache(id)}"

```


## cache memoize源码解析
### **cached**

```python
    def cached(
        self,
        timeout=None,
        key_prefix="view/%s",
        unless=None,
        forced_update=None,
        response_filter=None,
        query_string=False,
        hash_method=hashlib.md5,
    ):
        """..."""

        def decorator(f):
            @functools.wraps(f)
            def decorated_function(*args, **kwargs):...

                if not found:
                    rv = f(*args, **kwargs)

                    if response_filter is None or response_filter(rv):
                        try:
                            self.cache.set(
                                cache_key,
                                rv,
                                timeout=decorated_function.cache_timeout,
                            )
                        except Exception:
                            if self.app.debug:
                                raise
                            logger.exception(
                                "Exception possibly due to cache backend."
                            )
                return rv

            def make_cache_key(*args, **kwargs):...

            def _make_cache_key_query_string():...

            def _make_cache_key(args, kwargs, use_request):...

            decorated_function.uncached = f
            decorated_function.cache_timeout = timeout
            decorated_function.make_cache_key = make_cache_key

            return decorated_function

        return decorator
```


- cached是针对flask视图函数的缓存，使用过程中，我们主要关注以下几个参数：
	- timeout用于指定缓存失效的时间，设置0或-1则不失效
	- unless是一个函数，用于判断是否需要跳过缓存，可自定义实现
		

```python
def _make_cache_key(args, kwargs, use_request):
    if callable(key_prefix):
        cache_key = key_prefix()
    elif "%s" in key_prefix:
        if use_request:
            cache_key = key_prefix % request.path
        else:
            cache_key = key_prefix % url_for(f.__name__, **kwargs)
    else:
        cache_key = key_prefix

    return cache_key

# ```

- 默认情况下进入逻辑"elif "%s" in key_prefix:", 因此cached缓存只与requesturl相关，与视图函数的传参无关，`这里不太了解作者的实现意图，为什么不结合请求参数，且不太方便扩展或重写_make_cache_key`，若需要结合请求参数进行缓存，则就要对整个cached方便重写，存在大量冗余代码。具体的实现大概为通过flask.request.get_json()获取request_body，并结合业务对影响缓存的参数设置缓存。


```python
def _make_cache_key(args, kwargs, use_request):
    if callable(key_prefix):
        cache_key = key_prefix()
    elif "%s" in key_prefix:
        if use_request:
            cache_key = key_prefix % request.path
        else:
            cache_key = key_prefix % url_for(f.__name__, **kwargs)
    else:
        cache_key = key_prefix

    return cache_key
```
- 另外值得学习的是装饰器最后一层的写法，结合了面向对象的思想，设置关键的属性到函数对象，结合后续删除等操作，非常优雅非常pythonic。

### **memoize**
- memoize相较于cached的区别
	- cache_key实现不同，memoize对key进行了加密
	- 针对普通函数缓存结果使用
	- 支持区分函数参数的缓存，索引cached基于参数的缓存改造，可以参考memoize的实现

## 总结-如何写一个缓存框架
从flask-caching的实现看出，实现缓存，逻辑还是很清晰的：`判断是否跳过缓存，生成查询key，获取缓存数据`。
我们如果要实现缓存框架，大体需要考虑一下几方面：
- 支持使用不同的存储实例，**抽象缓存对象接口**，如`get，has，set`
- 支持区分参数缓存
- 支持缓存时效性
- 支持缓存数据加密
- 如果要结合业务，则最好再额外添加业务层，统一对缓存调用进行管理


## 使用缓存的注意事项（待补充）
- 缓存穿透
- 缓存雪崩
- 缓存集群