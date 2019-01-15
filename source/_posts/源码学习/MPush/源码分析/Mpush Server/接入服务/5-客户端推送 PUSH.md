---
title: 5-客户端推送 PUSH
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

接收客户端的PUSH请求，请求处理器走的SPI机制，业务实现由SPI扩展实现；
```java
//ConnectionServer#init()
messageDispatcher.register(Command.PUSH, PushHandlerFactory::create);
```

```java
public interface PushHandlerFactory extends Factory<MessageHandler> {
    static MessageHandler create() {
        return SpiLoader.load(PushHandlerFactory.class).get();
    }
}
```

```java
@Spi(order = 1)
public final class ClientPushHandler extends BaseMessageHandler<PushMessage> implements PushHandlerFactory {
    @Override
    public PushMessage decode(Packet packet, Connection connection) {
        return new PushMessage(packet, connection);
    }
    @Override
    public void handle(PushMessage message) {
        Logs.PUSH.info("receive client push message={}", message);

        if (message.autoAck()) {
            AckMessage.from(message).sendRaw();
            Logs.PUSH.info("send ack for push message={}", message);
        }
        //biz code write here
    }
    @Override
    public MessageHandler get() {
        return this;
    }
}
```


<br>
接入服务文章目录:

* [1-心跳 HEARTBEAT](../1-心跳 HEARTBEAT)
* [2-握手 HANDSHAKE](../2-握手 HANDSHAKE)
* [3-用户绑定-解绑 BIND-UNBIND](../3-用户绑定-解绑 BIND-UNBIND)
* [4-快速连接 FAST_CONNECT](../4-快速连接 FAST_CONNECT)
* <font color="red">5-客户端推送 PUSH</font>
* [6-消息确认 ACK](../6-消息确认 ACK)
* [7-HTTP代理 HTTP_PROXY](../7-HTTP代理 HTTP_PROXY)
