---
title: UserEventConsumer 用户事件订阅
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

用户在线列表的key为 `mp:oul:127.0.0.1`  
1、初始化UserManager  
踢人、清空在线用户列表、将用户添加到在线列表中、从在线列表中删除用户；  
统计在线用户数量、获取在线用户列表；  
2、订阅用户在线事件UserOnlineEvent  
将用户添加到在线列表中；  
发布MQ在线消息ONLINE_CHANNEL给订阅方；  
3、订阅用户离线事件UserOfflineEvent  
从在线列表中删除用户；  
发布MQ离线消息OFFLINE_CHANNEL给订阅方；  

UserEventConsumer.java
```java
package com.mpush.core.router;

import com.google.common.eventbus.AllowConcurrentEvents;
import com.google.common.eventbus.Subscribe;
import com.mpush.api.event.UserOfflineEvent;
import com.mpush.api.event.UserOnlineEvent;
import com.mpush.api.spi.common.MQClient;
import com.mpush.api.spi.common.MQClientFactory;
import com.mpush.common.router.RemoteRouterManager;
import com.mpush.common.user.UserManager;
import com.mpush.tools.event.EventConsumer;

import static com.mpush.api.event.Topics.OFFLINE_CHANNEL;
import static com.mpush.api.event.Topics.ONLINE_CHANNEL;

/**
 * Created by ohun on 2015/12/23.
 *
 * @author ohun@live.cn
 */
public final class UserEventConsumer extends EventConsumer {

    private final MQClient mqClient = MQClientFactory.create();

    private final UserManager userManager;

    public UserEventConsumer(RemoteRouterManager remoteRouterManager) {
        this.userManager = new UserManager(remoteRouterManager);
    }

    @Subscribe
    @AllowConcurrentEvents
    void on(UserOnlineEvent event) {
        userManager.addToOnlineList(event.getUserId());
        mqClient.publish(ONLINE_CHANNEL, event.getUserId());
    }

    @Subscribe
    @AllowConcurrentEvents
    void on(UserOfflineEvent event) {
        userManager.remFormOnlineList(event.getUserId());
        mqClient.publish(OFFLINE_CHANNEL, event.getUserId());
    }

    public UserManager getUserManager() {
        return userManager;
    }
}

```

UserManager.java
```java
/**
 * 在线列表是存在redis里的，服务被kill -9的时候，无法修改redis。
 * 查询全部在线列表的时候，要通过当前ZK里可用的机器来循环查询。
 * 每台机器的在线列表是分开存的，如果都存储在一起，某台机器挂了，反而不好处理。
 */
public final class UserManager {
    private static final Logger LOGGER = LoggerFactory.getLogger(UserManager.class);

    private final String onlineUserListKey = CacheKeys.getOnlineUserListKey(ConfigTools.getPublicIp());

    private final CacheManager cacheManager = CacheManagerFactory.create();

    private final MQClient mqClient = MQClientFactory.create();

    private final RemoteRouterManager remoteRouterManager;

    public UserManager(RemoteRouterManager remoteRouterManager) {
        this.remoteRouterManager = remoteRouterManager;
    }

    public void kickUser(String userId) {
        kickUser(userId, -1);
    }

    public void kickUser(String userId, int clientType) {
        Set<RemoteRouter> remoteRouters = remoteRouterManager.lookupAll(userId);
        if (remoteRouters != null) {
            for (RemoteRouter remoteRouter : remoteRouters) {
                ClientLocation location = remoteRouter.getRouteValue();
                if (clientType == -1 || location.getClientType() == clientType) {
                    MQKickRemoteMsg message = new MQKickRemoteMsg()
                            .setUserId(userId)
                            .setClientType(location.getClientType())
                            .setConnId(location.getConnId())
                            .setDeviceId(location.getDeviceId())
                            .setTargetServer(location.getHost())
                            .setTargetPort(location.getPort());
                    mqClient.publish(Constants.getKickChannel(location.getHostAndPort()), message);
                }
            }
        }
    }

    public void clearOnlineUserList() {
        cacheManager.del(onlineUserListKey);
    }

    public void addToOnlineList(String userId) {
        cacheManager.zAdd(onlineUserListKey, userId);
        LOGGER.info("user online {}", userId);
    }

    public void remFormOnlineList(String userId) {
        cacheManager.zRem(onlineUserListKey, userId);
        LOGGER.info("user offline {}", userId);
    }

    //在线用户数量
    public long getOnlineUserNum() {
        Long value = cacheManager.zCard(onlineUserListKey);
        return value == null ? 0 : value;
    }

    //在线用户数量
    public long getOnlineUserNum(String publicIP) {
        String online_key = CacheKeys.getOnlineUserListKey(publicIP);
        Long value = cacheManager.zCard(online_key);
        return value == null ? 0 : value;
    }

    //在线用户列表
    public List<String> getOnlineUserList(String publicIP, int start, int end) {
        String key = CacheKeys.getOnlineUserListKey(publicIP);
        return cacheManager.zrange(key, start, end, String.class);
    }
}

```
