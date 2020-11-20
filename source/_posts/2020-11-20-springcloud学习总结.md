---
title: springcloud学习总结
date: 2020-11-20 20:07:22
tags: spring cloud
categories: java
---

{%note info%}

微服务当道，咱也不能落下

{%endnote%}
<!--more-->



### 概述

- 微服务的概念就不多说了，网上随便搜。
- springcloud是一个为简化微服务开发而存在的框架。
- 在微服务架构大行其道的今天，springcloud的功能组件也是日新月异，所以这篇文章主要还是聊聊springcloud当下在各个子领域的优秀组件，希望能为理解微服务起到提纲挈领的作用。

---

### 功能组件

- 服务注册中心

    - Netflix Eureka（停更）

      - CAP的AP
      - 如果 Eureka 服务器检测到超过预期数量的注册客户端以一种不优雅的方式终止了连接，并且同时正在等待被驱逐，那么它们将进入自我保护模式。这样做是为了确保灾难性网络事件不会擦除eureka注册表数据，并将其向下传播到所有客户端
        - 任何客户端，如果连续3次心跳更新失败，那么它将被视为非正常终止，client将被剔除。当超过当前注册实例15%的客户端都处于这种状态，那么自我保护将被开启
        - 当自我保护开启以后，eureka服务器将停止剔除所有实例，直到
            - 它看到的心跳续借的数量回到了预期的阈值之上
            - 自我保护被禁用

    - Zookeeper

      - CAP的AP

    - Consul

      - CAP的AP

      - 功能

        - 服务发现
          - 提供HTTP和DNS两种发现方式
        - 健康监测
            - 支持多种协议，HTTP、TCP、Docker、Shell脚本定制化
        - KV存储
            - key , Value的存储方式
        - 多数据中心
            - Consul支持多数据中心
        - 可视化Web界面    

      - 容器启动

        ```docker
        docker run -d -p 8500:8500 -v /home/consul:/consul/data -e CONSUL_BIND_INTERFACE='eth0' --name=consul1 consul agent -server -bootstrap -ui -client='0.0.0.0'
        ```

        - agent: 表示启动 agent 进程
        - server: 表示 consul 为 server 模式
        - client: 表示 consul 为 client 模式
        - bootstrap: 表示这个节点是 Server-Leader
        - ui: 启动 Web UI, 默认端口 8500
        - node: 指定节点名称, 集群中节点名称唯一
        - client: 绑定客户端接口地址, 0.0.0.0表示所有地址都可以访问

    - Nacos（推荐）

      - 支持AP和CP切换
        - 一个更易于构建云原生应用的动态服务发现、配置管理和服务管理平台

- 客服端负载均衡
    - Netflix Ribbon（将停更）
      - nginx是集中式LB
      - ribbon是进程内，会在注册中心获取注册信息服务列表缓存到JVM，从而在本地实现RPC远程服务调用
      - ribbon = 负载均衡 + RestTemplate
    - LoadBalancer
    
- 服务调用
    - Netflix Feign（将停更）
      - springcloud在feign的基础上开发
      - feign是一个服务调用框架，集成了ribbon
      - 只需要定义服务绑定接口且以声明式的方法，优雅简单的实现服务调用
    - OpenFeign
      - Feign的接班人
      - 使用
        - 微服务接口 + 业务类@FeignClient + 主启动@EnableFeignClients（优雅如dubbo）
    
- 服务降级
    - Netflix Hystrix（将停更）
      - 服务降级 fallback
        - 可以在服务端和客户端实现，一般在客户端实现
        - 触发场景
            - 程序运行异常
            - 超时
            - 服务熔断触发服务降级
            - 线程池/信号量打满也会导致服务降级
      - 服务限流 flowlimit
      - 服务熔断 break 
          - 触发场景
              - 流量过大
          - 处理过程
              - 降级、熔断、恢复
          - 熔断类型
              - 熔断打开
                  - 请求不再进行调用当前服务，内部设置时钟一般为MTTR(平均故障处理时间)，当打开时长达到所设时钟则进入熔断状态
              - 熔断关闭
                  - 熔断关闭不会对服务进行熔断
              - 熔断半开
                  - 部分请求根据规则调用当前服务，如果请求成功且符合规则则认为当前服务恢复正常，关闭熔断
          - 断路器开启或者关闭的条件
              - 当满足一定阀值的时候（默认10秒内超过20个请求次数）
              - 当失败率达到一定的时候（默认10秒内超过50%请求失败）
              - 到达以上阀值，断路器将会开启
              - 当开启的时候，所有请求都不会进行转发
              - 一段时间之后（默认是5秒），这个时候断路器是半开状态，会让其中一个请求进行转发。如果成功，断路器会关闭，若失败，继续开启。重复4和5
    - resilience4j（国外较多）
    - sentienl（推荐）
      - 把流量作为切入点，从流量控制、熔断降级、系统负载保护等多个维度保护服务的稳定性
    
- 服务网关
    - Netflix Zuul（将停更）
    - Zuul2
    - gateway（推荐）
      - Spring Cloud Gateway 使用的Webflux中的reactor-netty响应式编程组件，底层使用了Netty通讯框架
      - 使用场景
        - 反向代理
        - 鉴权
        - 流量控制
        - 熔断
        - 日志监控
    
- 服务配置
    - Spring Cloud Config
    - Nacos（推荐）
    
- 服务总线
    
    - springcloud stream 
      - 屏蔽底层消息中间件的差异，降低切换版本，统一消息的编程模型
    
    - springcloud bus
        - 当前仅支持消息代理rabbitmq、kafka
        - 配合springcloud config实现配置自动刷新
    - Nacos

- 链路追踪

  - sleuth
    - 收集整理分布式链路数据
    - 结合zippkin使用

  - zippkin

    - 展示分布式链路数据

    - docker启动

      ```dockerdo c
      docker run -d -p 9411:9411 openzipkin/zipkin	
      ```

- 分布式事务

  - seata
    - 一个易于使用的高性能微服务分布式事务解决方案

---

### 总结

- 可以看出随着Netflix大部分组件的停更，springcloud也出现大换血，当随之出现了设计理念更加先进的springcloud alibaba，其涵盖了前者大部分的功能点，短短几年时间，springcloud的新老交替，更新迭代不可谓不快，但生命不息，学习不止，这便是程序员的宿命啊。