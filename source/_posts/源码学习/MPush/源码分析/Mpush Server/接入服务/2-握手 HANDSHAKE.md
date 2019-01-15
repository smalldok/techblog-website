---
title: 2-握手 HANDSHAKE
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
messageDispatcher.register(Command.HANDSHAKE, () -> new HandshakeHandler(mPushServer));
```
接收到客户端的Handshake请求，根据SessionContext中是否设置是否启用加密，分2种处理请求：
```java
//HandshakeHandler.java
@Override
public void handle(HandshakeMessage message) {
    if (message.getConnection().getSessionContext().isSecurity()) {
        doSecurity(message);
    } else {
        doInsecurity(message);
    }
}
```
启用加密的处理：  
加解密的交互细节，`站内搜索文章：深度进阶-加解密`  
握手成功，生成session并保存到redis中，用于快速重连；  
```java
private void doSecurity(HandshakeMessage message) {
    byte[] iv = message.iv;//AES密钥向量16位
    byte[] clientKey = message.clientKey;//客户端随机数16位
    byte[] serverKey = CipherBox.I.randomAESKey();//服务端随机数16位
    byte[] sessionKey = CipherBox.I.mixKey(clientKey, serverKey);//会话密钥16位

    //1.校验客户端消息字段
    if (Strings.isNullOrEmpty(message.deviceId)
            || iv.length != CipherBox.I.getAesKeyLength()
            || clientKey.length != CipherBox.I.getAesKeyLength()) {
        ErrorMessage.from(message).setReason("Param invalid").close();
        Logs.CONN.error("handshake failure, message={}, conn={}", message, message.getConnection());
        return;
    }

    //2.重复握手判断
    SessionContext context = message.getConnection().getSessionContext();
    if (message.deviceId.equals(context.deviceId)) {
        ErrorMessage.from(message).setErrorCode(REPEAT_HANDSHAKE).send();
        Logs.CONN.warn("handshake failure, repeat handshake, conn={}", message.getConnection());
        return;
    }

    //3.更换会话密钥RSA=>AES(clientKey)
    context.changeCipher(new AesCipher(clientKey, iv));

    //4.生成可复用session, 用于快速重连
    ReusableSession session = reusableSessionManager.genSession(context);

    //5.计算心跳时间
    int heartbeat = ConfigTools.getHeartbeat(message.minHeartbeat, message.maxHeartbeat);

    //6.响应握手成功消息
    HandshakeOkMessage
            .from(message)
            .setServerKey(serverKey)
            .setHeartbeat(heartbeat)
            .setSessionId(session.sessionId)
            .setExpireTime(session.expireTime)
            .send(f -> {
                        if (f.isSuccess()) {
                            //7.更换会话密钥AES(clientKey)=>AES(sessionKey)
                            context.changeCipher(new AesCipher(sessionKey, iv));
                            //8.保存client信息到当前连接
                            context.setOsName(message.osName)
                                    .setOsVersion(message.osVersion)
                                    .setClientVersion(message.clientVersion)
                                    .setDeviceId(message.deviceId)
                                    .setHeartbeat(heartbeat);

                            //9.保存可复用session到Redis, 用于快速重连
                            reusableSessionManager.cacheSession(session);

                            Logs.CONN.info("handshake success, conn={}", message.getConnection());
                        } else {
                            Logs.CONN.info("handshake failure, conn={}", message.getConnection(), f.cause());
                        }
                    }
            );
}
```
没有启用加密的处理：
```java
private void doInsecurity(HandshakeMessage message) {
        //1.校验客户端消息字段
        if (Strings.isNullOrEmpty(message.deviceId)) {
            ErrorMessage.from(message).setReason("Param invalid").close();
            Logs.CONN.error("handshake failure, message={}, conn={}", message, message.getConnection());
            return;
        }

        //2.重复握手判断
        SessionContext context = message.getConnection().getSessionContext();
        if (message.deviceId.equals(context.deviceId)) {
            ErrorMessage.from(message).setErrorCode(REPEAT_HANDSHAKE).send();
            Logs.CONN.warn("handshake failure, repeat handshake, conn={}", message.getConnection());
            return;
        }

        //6.响应握手成功消息
        HandshakeOkMessage.from(message).send();

        //8.保存client信息到当前连接
        context.setOsName(message.osName)
                .setOsVersion(message.osVersion)
                .setClientVersion(message.clientVersion)
                .setDeviceId(message.deviceId)
                .setHeartbeat(Integer.MAX_VALUE);

        Logs.CONN.info("handshake success, conn={}", message.getConnection());

    }
}
```
session管理
```java
public final class ReusableSessionManager {
    private final int expiredTime = CC.mp.core.session_expired_time;
    private final CacheManager cacheManager = CacheManagerFactory.create();

    public boolean cacheSession(ReusableSession session) {
        String key = CacheKeys.getSessionKey(session.sessionId);
        cacheManager.set(key, ReusableSession.encode(session.context), expiredTime);
        return true;
    }

    public ReusableSession querySession(String sessionId) {
        String key = CacheKeys.getSessionKey(sessionId);
        String value = cacheManager.get(key, String.class);
        if (Strings.isBlank(value)) return null;
        return ReusableSession.decode(value);
    }

    public ReusableSession genSession(SessionContext context) {
        long now = System.currentTimeMillis();
        ReusableSession session = new ReusableSession();
        session.context = context;
        session.sessionId = MD5Utils.encrypt(context.deviceId + now);
        session.expireTime = now + expiredTime * 1000;
        return session;
    }
}
```
是否加密的设置代码：
```java
public final class ConnectionServer extends NettyTCPServer {
    public ConnectionServer(MPushServer mPushServer) {
        this.channelHandler = new ServerChannelHandler(true, connectionManager, messageDispatcher);
    }
}
```

```java
@ChannelHandler.Sharable
public final class ServerChannelHandler extends ChannelInboundHandlerAdapter {
    private final boolean security; //是否启用加密
    public ServerChannelHandler(boolean security, ConnectionManager connectionManager, PacketReceiver receiver) {
        this.security = security;
    }
    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        Logs.CONN.info("client connected conn={}", ctx.channel());
        Connection connection = new NettyConnection();
        connection.init(ctx.channel(), security);
        connectionManager.add(connection);
    }
}
```

```java
public final class NettyConnection implements Connection, ChannelFutureListener {
    private static final Logger LOGGER = LoggerFactory.getLogger(NettyConnection.class);
    private SessionContext context;
    private Channel channel;
    private volatile byte status = STATUS_NEW;
    private long lastReadTime;
    private long lastWriteTime;

    @Override
    public void init(Channel channel, boolean security) {
        this.channel = channel;
        this.context = new SessionContext();
        this.lastReadTime = System.currentTimeMillis();
        this.status = STATUS_CONNECTED;
        if (security) {
            this.context.changeCipher(RsaCipherFactory.create());
        }
    }
}
```

<br>
接入服务文章目录:

* [1-心跳 HEARTBEAT](../1-心跳 HEARTBEAT)
* <font color="red">2-握手 HANDSHAKE</font>
* [3-用户绑定-解绑 BIND-UNBIND](../3-用户绑定-解绑 BIND-UNBIND)
* [4-快速连接 FAST_CONNECT](../4-快速连接 FAST_CONNECT)
* [5-客户端推送 PUSH](../5-客户端推送 PUSH)
* [6-消息确认 ACK](../6-消息确认 ACK)
* [7-HTTP代理 HTTP_PROXY](../7-HTTP代理 HTTP_PROXY)
