---
title: RemoteRouterManager 远程路由管理器
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

1、通过SPI，找到mpush-cache模块中CacheManagerFactory接口的实现类RedisCacheManagerFactory，得到RedisManager实例；  
2、远程路由信息的添加、删除、获取；  
3、订阅连接关闭事件ConnectionCloseEvent，将远程路由信息修改为离线(connId=null)；  

```java
public class RemoteRouterManager extends EventConsumer implements RouterManager<RemoteRouter> {
    public static final Logger LOGGER = LoggerFactory.getLogger(RemoteRouterManager.class);

    private final CacheManager cacheManager = CacheManagerFactory.create();

    @Override
    public RemoteRouter register(String userId, RemoteRouter router) {
        String key = CacheKeys.getUserRouteKey(userId);
        String field = Integer.toString(router.getRouteValue().getClientType());
        ClientLocation old = cacheManager.hget(key, field, ClientLocation.class);
        cacheManager.hset(key, field, router.getRouteValue().toJson());
        LOGGER.info("register remote router success userId={}, newRouter={}, oldRoute={}", userId, router, old);
        return old == null ? null : new RemoteRouter(old);
    }

    /**
     * 目前的实现方式是非原子操作(get:set)，可能会有并发问题，虽然概率很低
     * 后续考虑采用lua脚本，实现原子操作
     *
     * @param userId     用户ID
     * @param clientType 客户端类型
     * @return 删除路由是否成功
     */
    @Override
    public boolean unRegister(String userId, int clientType) {
        String key = CacheKeys.getUserRouteKey(userId);
        String field = Integer.toString(clientType);
        ClientLocation location = cacheManager.hget(key, field, ClientLocation.class);
        if (location == null || location.isOffline()) return true;
        cacheManager.hset(key, field, location.offline().toJson());
        LOGGER.info("unRegister remote router success userId={}, route={}", userId, location);
        return true;
    }

    @Override
    public Set<RemoteRouter> lookupAll(String userId) {
        String key = CacheKeys.getUserRouteKey(userId);
        Map<String, ClientLocation> values = cacheManager.hgetAll(key, ClientLocation.class);
        if (values == null || values.isEmpty()) return Collections.emptySet();
        return values.values().stream().map(RemoteRouter::new).collect(Collectors.toSet());
    }

    @Override
    public RemoteRouter lookup(String userId, int clientType) {
        String key = CacheKeys.getUserRouteKey(userId);
        String field = Integer.toString(clientType);
        ClientLocation location = cacheManager.hget(key, field, ClientLocation.class);
        LOGGER.info("lookup remote router userId={}, router={}", userId, location);
        return location == null ? null : new RemoteRouter(location);
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
        String key = CacheKeys.getUserRouteKey(userId);
        String field = Integer.toString(context.getClientType());
        ClientLocation location = cacheManager.hget(key, field, ClientLocation.class);
        if (location == null || location.isOffline()) return;

        String connId = connection.getId();
        //2.检测下，是否是同一个链接, 如果客户端重连，老的路由会被新的链接覆盖
        if (connId.equals(location.getConnId())) {
            cacheManager.hset(key, field, location.offline().toJson());
            LOGGER.info("clean disconnected remote route, userId={}, route={}", userId, location);
        } else {
            LOGGER.info("clean disconnected remote route, not clean:userId={}, route={}", userId, location);
        }
    }
}

```

RemoteRouter.java
```java
public final class RemoteRouter implements Router<ClientLocation> {
    private final ClientLocation clientLocation;
    public RemoteRouter(ClientLocation clientLocation) {
        this.clientLocation = clientLocation;
    }
    public boolean isOnline(){
        return clientLocation.isOnline();
    }
    public boolean isOffline(){
        return clientLocation.isOffline();
    }
    @Override
    public ClientLocation getRouteValue() {
        return clientLocation;
    }
    @Override
    public RouterType getRouteType() {
        return RouterType.REMOTE;
    }
    @Override
    public String toString() {
        return "RemoteRouter{" + clientLocation + '}';
    }
}

```
