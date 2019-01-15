---
title: SPI定制化
comments: true
tags:
  - 源码
  - 消息推送
categories:
  - 源码学习
  - MPush
  - 架构分析
abbrlink: 27765
date: 2016-03-01 21:00:00
---

# 什么是SPI
建议直接百度 java spi, 官方介绍: http://docs.oracle.com/javase/tutorial/sound/SPI-intro.html

# mpush 使用SPI的目的
肯定是为了扩展性，在不想修改源码的情况下，去替换系统原有的实现，代价最小也最灵活。

# mpush目前支持的SPI列表
所有受支持的SPI都在mpuhs-api模块的com.mpush.api.spi包下面。  
* `ServerEventListenerFactory` 监听Server 产生的相关事件
* `BindValidatorFactory` 绑定用户身份是校验用户身份是否合法
* `ClientClassifierFactory` 同一个用户多设备登录互踢策略，默认Android和ios会互踢
* `PushHandlerFactory` 接收客户端发送到服务端的上行推送消息
* `ExecutorFactory` 整个系统的线程池工厂，可以按自己的策略创建线程池
* `CacheManagerFactory` 系统使用的缓存实现，默认是redis，支持自定义
* `MQClientFactory` 系统使用的MQ实现，默认是redis的pub/sub，支持自定义
* `ServiceRegistryFactory` 服务注册组件，默认实现是zookeeper，支持自定义
* `ServiceDiscoveryFactory` 服务发现组件，默认实现是zookeeper，支持自定义
* `PushListenerFactory` 消息来源自定义，默认使用Gateway模块，支持自定义，比如使用MQ

# 开发流程
* 新建一个maven工程，并加入mpush-api模块的依赖
* 实现要定制的接口和其对应的Factory
* 在resources目录下创建META-INF.services目录
* 创建名字为`com.mpush.api.spi.xxx.XXXFactory`的文件，即对应的接口名
* 文件的内容为实现类的全名(packageName+className)
* 通过mvn打成jar包，并把其放到mpush/lib目录
* 重启mpush server 就会优先加载用户提供的实现来覆盖原有的默认实现
