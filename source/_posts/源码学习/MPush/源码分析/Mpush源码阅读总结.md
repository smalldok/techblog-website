---
title: Mpush源码阅读总结
comments: true
tags:
  - 源码
  - 消息推送
categories:
  - 源码学习
  - MPush
  - 源码分析
abbrlink: 20000
date: 2016-03-01 21:00:00
---



# 总结
* 通盘阅读其源码，可以了解`端到云`、`云到端`的消息`推送流程`和`实现细节`；
* 有客户端DEMO(安卓/IOS/WEB端) 、服务端DEMO、业务端DEMO
* 实现了`TCP服务`、`websocket服务`、`HTTP代理服务`(客户端通过http下发消息)
* 定义了统一的`生命周期组件`，统一组件的初始化、启动、关闭  
例如：NettyTCPServer extends BaseService
* `SPI扩展`，支持指定名称的单个加载、多个加载时排序取第一个
* `流控`  
    * `GlobalFlowControl`   #单进程全局流控，对MPNS发送给网关的流控(单进程)  
    * `RedisFlowControl`    #redis   流控(多进程)，用于限制广播消息到客户端的数量（这里代码实现不安全）
    * `FastFlowControl`     #快速流控(单进程)，用于限制广播消息到客户端的数量（这里代码实现不安全）  
    * `ExactFlowControl`    #分段流控，分段统计，把1s 分成 100份，10ms一份，限制10ms内允许的最大数量  
* 配置集中管理
* `监控`，实现了JVM、各个线程池指标的监控
* `性能监控`，利用Profiler做方法耗时的埋点打印
* `时间线`，通过TimeLine，可以清晰知道msg流转到哪个组件，有点像trace，只不过它是单进程的
* `线程池`，定义了5种线程池，包括MQ/event-bus/push-client-timer/push-task-timer/ack-timer
* `事件总线`，使用Guava的EventBus，方便其他组件订阅建连、断连、异常事件

# 不足
## 网络组件
不够抽象，与消息推送的实现逻辑混合在一起，使得结构不清晰；
关于网络组件抽象这一块，可以参考下SOFA-Bolt的实现；

> SOFABolt 是蚂蚁金融服务集团开发的一套基于 Netty 实现的网络通信框架。  
> * 为了让 Java 程序员能将更多的精力放在基于网络通信的业务逻辑实现上，而不是过多的纠结于网络底层 NIO 的实现以及处理难以调试的网络问题，Netty 应运而生。
> * 为了让中间件开发者能将更多的精力放在产品功能特性实现上，而不是重复地一遍遍制造通信框架的轮子，SOFABolt 应运而生。  

> 该产品已经运用在了蚂蚁中间件的微服务 (SOFARPC)、消息中心、分布式事务、分布式开关、以及配置中心等众多产品上  
https://github.com/alipay/sofa-bolt

## 通信模型与超时控制
通信模型没有抽象出来(oneway/sync/future/callback)

超时控制用了netty的HashedWheelTimer和SchedulerExecutor调度线程池这2种实现；  
目测之所以采用SchedulerExecutor调度线程池，是因为在客户端代码中只想依赖JDK实现；

可以在站内搜索文章《通信模型与超时控制》

## 配置
集中在一个类里面，虽然集中管理比较方便，但始终感觉不太优雅；  
不支持扩展通过配置中心动态修改配置；

## 流控
代码实现不太安全
