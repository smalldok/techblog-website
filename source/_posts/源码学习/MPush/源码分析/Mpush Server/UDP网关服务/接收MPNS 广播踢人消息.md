---
title: 接收MPNS 广播踢人消息
comments: true
tags:
  - 源码
  - 消息推送
categories:
  - 源码学习
  - MPush
  - 源码分析
  - Mpush Server
  - UDP网关服务
abbrlink: 25851
date: 2016-03-01 21:00:00
---

GatewayUDPConnector服务，接收MPNS的消息种类如下：
```java
//GatewayUDPConnector#init()
//PUSH消息
messageDispatcher.register(Command.GATEWAY_PUSH, () -> new GatewayPushHandler(mPushServer.getPushCenter()));
//踢人消息
messageDispatcher.register(Command.GATEWAY_KICK, () -> new GatewayKickUserHandler(mPushServer.getRouterCenter()));

```

处理踢人消息：
```java
public final class GatewayKickUserHandler extends BaseMessageHandler<GatewayKickUserMessage> {
    private final RouterCenter routerCenter;
    public GatewayKickUserHandler(RouterCenter routerCenter) {
        this.routerCenter = routerCenter;
    }
    @Override
    public GatewayKickUserMessage decode(Packet packet, Connection connection) {
        return new GatewayKickUserMessage(packet, connection);
    }
    @Override
    public void handle(GatewayKickUserMessage message) {
        routerCenter.getRouterChangeListener().onReceiveKickRemoteMsg(message);
    }
}

```

RouterChangeListener.java
```java
/**
 * 处理远程机器发送的踢人广播.
 * <p>
 * 一台机器发送广播所有的机器都能收到，
 * 包括发送广播的机器，所有要做一次过滤
 *
 * @param msg
 */
public void onReceiveKickRemoteMsg(KickRemoteMsg msg) {
    //1.如果当前机器不是目标机器，直接忽略
    if (!mPushServer.isTargetMachine(msg.getTargetServer(), msg.getTargetPort())) {
        Logs.CONN.error("receive kick remote msg, target server error, localIp={}, msg={}", ConfigTools.getLocalIp(), msg);
        return;
    }

    //2.查询本地路由，找到要被踢下线的链接，并删除该本地路由
    String userId = msg.getUserId();
    int clientType = msg.getClientType();
    LocalRouterManager localRouterManager = mPushServer.getRouterCenter().getLocalRouterManager();
    LocalRouter localRouter = localRouterManager.lookup(userId, clientType);
    if (localRouter != null) {
        Logs.CONN.info("receive kick remote msg, msg={}", msg);
        if (localRouter.getRouteValue().getId().equals(msg.getConnId())) {//二次校验，防止误杀
            //fix 0.8.1 踢人的时候不再主动删除路由信息，只发踢人消息到客户端，路由信息有客户端主动解绑的时候再处理。
            //2.1删除本地路由信息
            //localRouterManager.unRegister(userId, clientType);
            //2.2发送踢人消息到客户端
            sendKickUserMessage2Client(userId, localRouter);
        } else {
            Logs.CONN.warn("kick router failure target connId not match, localRouter={}, msg={}", localRouter, msg);
        }
    } else {
        Logs.CONN.warn("kick router failure can't find local router, msg={}", msg);
    }
}

/**
 * 发送踢人消息到客户端
 *
 * @param userId 当前用户
 * @param router 本地路由信息
 */
private void sendKickUserMessage2Client(final String userId, final LocalRouter router) {
    Connection connection = router.getRouteValue();
    SessionContext context = connection.getSessionContext();
    KickUserMessage message = KickUserMessage.build(connection);
    message.deviceId = context.deviceId;
    message.userId = userId;
    message.send(future -> {
        if (future.isSuccess()) {
            Logs.CONN.info("kick local connection success, userId={}, router={}, conn={}", userId, router, connection);
        } else {
            Logs.CONN.warn("kick local connection failure, userId={}, router={}, conn={}", userId, router, connection);
        }
    });
}

```
