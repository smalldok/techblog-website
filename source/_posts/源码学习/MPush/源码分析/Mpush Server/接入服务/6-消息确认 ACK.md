---
title: 6-消息确认 ACK
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

接收到客户端的ACK响应
```java
//ConnectionServer#init()
messageDispatcher.register(Command.ACK, () -> new AckHandler(mPushServer));
```

```java
public final class AckHandler extends BaseMessageHandler<AckMessage> {
    private final AckTaskQueue ackTaskQueue;
    public AckHandler(MPushServer mPushServer) {
        this.ackTaskQueue = mPushServer.getPushCenter().getAckTaskQueue();
    }
    @Override
    public AckMessage decode(Packet packet, Connection connection) {
        return new AckMessage(packet, connection);
    }
    @Override
    public void handle(AckMessage message) {
        AckTask task = ackTaskQueue.getAndRemove(message.getSessionId());
        if (task == null) {//ack 超时了
            Logs.PUSH.info("receive client ack, but task timeout message={}", message);
            return;
        }
        task.onResponse();//成功收到客户的ACK响应
    }
}
```


<br>
接入服务文章目录:

* [1-心跳 HEARTBEAT](../1-心跳 HEARTBEAT)
* [2-握手 HANDSHAKE](../2-握手 HANDSHAKE)
* [3-用户绑定-解绑 BIND-UNBIND](../3-用户绑定-解绑 BIND-UNBIND)
* [4-快速连接 FAST_CONNECT](../4-快速连接 FAST_CONNECT)
* [5-客户端推送 PUSH](../5-客户端推送 PUSH)
* <font color="red">6-消息确认 ACK</font>
* [7-HTTP代理 HTTP_PROXY](../7-HTTP代理 HTTP_PROXY)
