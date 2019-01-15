---
title: LocalRouterManager 本地路由管理器
comments: true
tags:
  - 源码
  - 消息推送
categories:
  - 源码学习
  - MPush
  - 源码分析
  - Mpush Server
  - 路由中心
abbrlink: 25851
date: 2016-03-01 21:00:00
---

1、本地路由信息的添加、删除、获取；  
2、订阅连接关闭事件ConnectionCloseEvent，删除本地路由，发布离线事件UserOfflineEvent；  

```java
package com.mpush.core.router;

import com.google.common.eventbus.AllowConcurrentEvents;
import com.google.common.eventbus.Subscribe;
import com.mpush.api.connection.Connection;
import com.mpush.api.connection.SessionContext;
import com.mpush.api.event.ConnectionCloseEvent;
import com.mpush.api.event.UserOfflineEvent;
import com.mpush.api.router.RouterManager;
import com.mpush.tools.event.EventBus;
import com.mpush.tools.event.EventConsumer;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.HashMap;
import java.util.HashSet;
import java.util.Map;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;

/**
 * Created by ohun on 2015/12/23.
 *
 * @author ohun@live.cn
 */
public final class LocalRouterManager extends EventConsumer implements RouterManager<LocalRouter> {
    private static final Logger LOGGER = LoggerFactory.getLogger(LocalRouterManager.class);
    private static final Map<Integer, LocalRouter> EMPTY = new HashMap<>(0);

    /**
     * 本地路由表
     */
    private final Map<String, Map<Integer, LocalRouter>> routers = new ConcurrentHashMap<>();

    //添加本地路由
    @Override
    public LocalRouter register(String userId, LocalRouter router) {
        LOGGER.info("register local router success userId={}, router={}", userId, router);
        return routers.computeIfAbsent(userId, s -> new HashMap<>(1)).put(router.getClientType(), router);
    }
    //删除本地路由
    @Override
    public boolean unRegister(String userId, int clientType) {
        LocalRouter router = routers.getOrDefault(userId, EMPTY).remove(clientType);
        LOGGER.info("unRegister local router success userId={}, router={}", userId, router);
        return true;
    }
    //查找所有本地路由
    @Override
    public Set<LocalRouter> lookupAll(String userId) {
        return new HashSet<>(routers.getOrDefault(userId, EMPTY).values());
    }
    //查找userId和clientType对应的本地路由
    @Override
    public LocalRouter lookup(String userId, int clientType) {
        LocalRouter router = routers.getOrDefault(userId, EMPTY).get(clientType);
        LOGGER.info("lookup local router userId={}, router={}", userId, router);
        return router;
    }
    //返回本地路由表
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
        //获取userId和clientType对应的本地路由
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

LocalRouter.java
```java
package com.mpush.core.router;

import com.mpush.api.connection.Connection;
import com.mpush.api.router.Router;

/**
 * Created by ohun on 2015/12/23.
 *
 * @author ohun@live.cn
 */
public final class LocalRouter implements Router<Connection> {
    private final Connection connection;

    public LocalRouter(Connection connection) {
        this.connection = connection;
    }

    public int getClientType() {
        return connection.getSessionContext().getClientType();
    }

    @Override
    public Connection getRouteValue() {
        return connection;
    }

    @Override
    public RouterType getRouteType() {
        return RouterType.LOCAL;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;

        LocalRouter that = (LocalRouter) o;

        return getClientType() == that.getClientType();

    }

    @Override
    public int hashCode() {
        return Integer.hashCode(getClientType());
    }

    @Override
    public String toString() {
        return "LocalRouter{" + connection + '}';
    }
}

```
