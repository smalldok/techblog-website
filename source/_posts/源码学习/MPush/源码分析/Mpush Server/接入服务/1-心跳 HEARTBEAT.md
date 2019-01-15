---
title: 1-心跳 HEARTBEAT
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

接收消息，解码协议、解码body内容，`站内搜索文章: 编码解码`

客户端发送心跳：  
场景一：  
>当与服务端建立连接时(ConnClientChannelHandler#channelActive)，发送握手或者快速连接请求；  
在接收到握手或者快速连接响应时(ConnClientChannelHandler#channelRead)，发送心跳请求；  

场景二：  
> 当网络断开时不主动关闭连接，而是尝试发送一次心跳检测，如果能收到响应，说明网络短时间内又恢复了，否则就断开连接，等待网络恢复并重建连接。  
见mpush-client-java工程 MPushClient#onNetStateChange()方法；

服务端接收处理心跳：
```java
//ConnectionServer#init()
messageDispatcher.register(Command.HEARTBEAT, HeartBeatHandler::new);
```

```java
public final class HeartBeatHandler implements MessageHandler {
    @Override
    public void handle(Packet packet, Connection connection) {
        connection.send(packet);//ping -> pong
        Logs.HB.info("ping -> pong, {}", connection);
    }
}
```

<br>
接入服务文章目录:

* <font color="red">1-心跳 HEARTBEAT</font>
* [2-握手 HANDSHAKE](../2-握手 HANDSHAKE)
* [3-用户绑定-解绑 BIND-UNBIND](../3-用户绑定-解绑 BIND-UNBIND)
* [4-快速连接 FAST_CONNECT](../4-快速连接 FAST_CONNECT)
* [5-客户端推送 PUSH](../5-客户端推送 PUSH)
* [6-消息确认 ACK](../6-消息确认 ACK)
* [7-HTTP代理 HTTP_PROXY](../7-HTTP代理 HTTP_PROXY)
