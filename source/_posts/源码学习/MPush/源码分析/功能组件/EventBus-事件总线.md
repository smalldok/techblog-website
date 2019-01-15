---
title: EventBus-事件总线
comments: true
tags:
  - 源码
  - 消息推送
categories:
  - 源码学习
  - MPush
  - 源码分析
  - 功能组件
abbrlink: 20000
date: 2016-03-01 21:00:00
---

# 介绍
一个内存级别的异步事件总线服务，实现了简单的生产-消费者模式；  
EventBus是Guava框架对观察者模式的一种实现，使用EventBus可以很简洁的实现事件注册监听和消费。  

使用场景：
* MPUSH中各事件的发布、订阅；
* Elastic-Job任务执行和任务轨迹记录

new EventBus();
new AsyncEventBus();
* post(new xxxEvent()) 发布事件
* register(this) 注册、订阅事件
@Subscribe
@AllowConcurrentEvents
* unregister(this) 取消订阅

第一步：哪个类需要订阅，首先的注册，然后订阅
```java
// 注册
EventBus.register(this);

// 订阅事件
@Subscribe
@AllowConcurrentEvents
void on(ConnectionCloseEvent event) {
    ....
}

@Subscribe
@AllowConcurrentEvents
void on(xxxxEvent event) {
    ....
}

```
第二步：发布事件
```java
EventBus.post(new UserOnlineEvent(message.getConnection(), message.userId));
```
参考：https://blog.csdn.net/fanhenghui/article/details/51459273

# MPUSH源码实现
MPushClient.java 创建EventBus
```java
public MPushClient() {
    monitorService = new MonitorService();
    EventBus.create(monitorService.getThreadPoolManager().getEventBusExecutor());
}
```

EventBus.java
```java
package com.mpush.tools.event;

import com.google.common.eventbus.AsyncEventBus;
import com.mpush.api.event.Event;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.concurrent.Executor;

/**
 * Created by ohun on 2015/12/29.
 *
 * @author ohun@live.cn
 */
public class EventBus {
    private static final Logger LOGGER = LoggerFactory.getLogger(EventBus.class);
    private static com.google.common.eventbus.EventBus eventBus;

    public static void create(Executor executor) {
        eventBus = new AsyncEventBus(executor, (exception, context)
                -> LOGGER.error("event bus subscriber ex", exception));
    }

    public static void post(Event event) {
        eventBus.post(event);
    }

    public static void register(Object bean) {
        eventBus.register(bean);
    }

    public static void unregister(Object bean) {
        eventBus.unregister(bean);
    }

}
```

EventConsumer.java 父类中注册，子类中可以订阅事件
```java
package com.mpush.tools.event;
public abstract class EventConsumer {
    public EventConsumer() {
        EventBus.register(this);
    }
}
```

```java
public final class LocalRouterManager extends EventConsumer implements RouterManager<LocalRouter> {
    private static final Logger LOGGER = LoggerFactory.getLogger(LocalRouterManager.class);
    private static final Map<Integer, LocalRouter> EMPTY = new HashMap<>(0);

    /**
     * 本地路由表
     */
    private final Map<String, Map<Integer, LocalRouter>> routers = new ConcurrentHashMap<>();

    @Override
    public LocalRouter register(String userId, LocalRouter router) {
        LOGGER.info("register local router success userId={}, router={}", userId, router);
        return routers.computeIfAbsent(userId, s -> new HashMap<>(1)).put(router.getClientType(), router);
    }

    @Override
    public boolean unRegister(String userId, int clientType) {
        LocalRouter router = routers.getOrDefault(userId, EMPTY).remove(clientType);
        LOGGER.info("unRegister local router success userId={}, router={}", userId, router);
        return true;
    }

    @Override
    public Set<LocalRouter> lookupAll(String userId) {
        return new HashSet<>(routers.getOrDefault(userId, EMPTY).values());
    }

    @Override
    public LocalRouter lookup(String userId, int clientType) {
        LocalRouter router = routers.getOrDefault(userId, EMPTY).get(clientType);
        LOGGER.info("lookup local router userId={}, router={}", userId, router);
        return router;
    }

    public Map<String, Map<Integer, LocalRouter>> routers() {
        return routers;
    }

    /**
     * 监听链接关闭事件，清理失效的路由
     *
     * @param event
     */
    @Subscribe
    @AllowConcurrentEvents
    void on(ConnectionCloseEvent event) {
        Connection connection = event.connection;
        if (connection == null) return;
        SessionContext context = connection.getSessionContext();

        String userId = context.userId;
        if (userId == null) return;

        int clientType = context.getClientType();
        LocalRouter localRouter = routers.getOrDefault(userId, EMPTY).get(clientType);
        if (localRouter == null) return;

        String connId = connection.getId();
        //2.检测下，是否是同一个链接, 如果客户端重连，老的路由会被新的链接覆盖
        if (connId.equals(localRouter.getRouteValue().getId())) {
            //3. 删除路由
            routers.getOrDefault(userId, EMPTY).remove(clientType);
            //4. 发送用户下线事件, 只有老的路由存在的情况下才发送，因为有可能又用户重连了，而老的链接又是在新连接之后才断开的
            //这个时候就会有问题，会导致用户变成下线状态，实际用户应该是在线的。
            EventBus.post(new UserOfflineEvent(event.connection, userId));
            LOGGER.info("clean disconnected local route, userId={}, route={}", userId, localRouter);
        } else { //如果不相等，则log一下
            LOGGER.info("clean disconnected local route, not clean:userId={}, route={}", userId, localRouter);
        }
    }
}
```

<br>
 功能组件文章目录:

* [CC-配置中心](../CC-配置中心)
* **[EventBus-事件总线](../EventBus-事件总线)**
* [FlowControl-流控](../FlowControl-流控)
* [JVMUtil](../JVMUtil)
* [Logs](../Logs)
* [Profiler-统计方法或者线程执行时间](../Profiler-统计方法或者线程执行时间)
* [Profiler入门](../Profiler入门)
* [SPI机制](../SPI机制)
* [TimeLine-时间线](../TimeLine-时间线)
* [服务启动监听](../服务启动监听)
* [监控](../监控)
* [通信模型与超时控制](../通信模型与超时控制)
* [线程池](../线程池)
* [状态判断-位运算](../状态判断-位运算)
