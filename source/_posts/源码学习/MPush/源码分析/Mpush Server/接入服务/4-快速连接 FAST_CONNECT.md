---
title: 4-快速连接 FAST_CONNECT
comments: true
tags:
  - 源码
  - 消息推送
categories:
  - 源码学习
  - MPush
  - 源码分析
  - Mpush Server
  - 接入服务
abbrlink: 20000
date: 2016-03-01 21:00:00
---

```java
//ConnectionServer#init()
messageDispatcher.register(Command.FAST_CONNECT, () -> new FastConnectHandler(mPushServer));
```

```java
public final class FastConnectHandler extends BaseMessageHandler<FastConnectMessage> {
    private final ReusableSessionManager reusableSessionManager;

    public FastConnectHandler(MPushServer mPushServer) {
        this.reusableSessionManager = mPushServer.getReusableSessionManager();
    }

    @Override
    public FastConnectMessage decode(Packet packet, Connection connection) {
        return new FastConnectMessage(packet, connection);
    }

    @Override
    public void handle(FastConnectMessage message) {
        //从缓存中心查询session
        Profiler.enter("time cost on [query session]");
        ReusableSession session = reusableSessionManager.querySession(message.sessionId);
        Profiler.release();
        if (session == null) {
            //1.没查到说明session已经失效了
            ErrorMessage.from(message).setErrorCode(SESSION_EXPIRED).send();
            Logs.CONN.warn("fast connect failure, session is expired, sessionId={}, deviceId={}, conn={}"
                    , message.sessionId, message.deviceId, message.getConnection().getChannel());
        } else if (!session.context.deviceId.equals(message.deviceId)) {
            //2.非法的设备, 当前设备不是上次生成session时的设备
            ErrorMessage.from(message).setErrorCode(INVALID_DEVICE).send();
            Logs.CONN.warn("fast connect failure, not the same device, deviceId={}, session={}, conn={}"
                    , message.deviceId, session.context, message.getConnection().getChannel());
        } else {
            //3.校验成功，重新计算心跳，完成快速重连
            int heartbeat = ConfigTools.getHeartbeat(message.minHeartbeat, message.maxHeartbeat);
            session.context.setHeartbeat(heartbeat);

            Profiler.enter("time cost on [send FastConnectOkMessage]");
            FastConnectOkMessage
                    .from(message)
                    .setHeartbeat(heartbeat)
                    .sendRaw(f -> {
                        if (f.isSuccess()) {
                            //4. 恢复缓存的会话信息(包含会话密钥等)
                            message.getConnection().setSessionContext(session.context);
                            Logs.CONN.info("fast connect success, session={}, conn={}", session.context, message.getConnection().getChannel());
                        } else {
                            Logs.CONN.info("fast connect failure, session={}, conn={}", session.context, message.getConnection().getChannel());
                        }
                    });

            Profiler.release();
        }
    }
}

```

<br>
接入服务文章目录:

* [1-心跳 HEARTBEAT](../1-心跳 HEARTBEAT)
* [2-握手 HANDSHAKE](../2-握手 HANDSHAKE)
* [3-用户绑定-解绑 BIND-UNBIND](../3-用户绑定-解绑 BIND-UNBIND)
* <font color="red">4-快速连接 FAST_CONNECT</font>
* [5-客户端推送 PUSH](../5-客户端推送 PUSH)
* [6-消息确认 ACK](../6-消息确认 ACK)
* [7-HTTP代理 HTTP_PROXY](../7-HTTP代理 HTTP_PROXY)
