---
title: admin server测试
comments: true
tags:
  - 源码
  - 消息推送
categories:
  - 源码学习
  - MPush
  - 源码分析
  - Mpush Server
  - 控制台服务
abbrlink: 25851
date: 2016-03-01 21:00:00
---

启动mpush服务  
运行ServerTestMain#testServer()，控制台日志会打印如下：
```java
11:47:00.097 - [mp-admin-boss-6-1] INFO  - com.mpush.core.server.AdminServer - server start success on:3002
```

telnet连接到mpush admin控制台
```java
➜  ~ telnet 172.16.177.134 3002
Trying 172.16.177.134...
Connected to 172.16.177.134.
Escape character is '^]'.
Welcome to MPush Console [172.16.177.134]!
since 2018-12-26T11:47:00.087 has running 0(h)

It is Wed Dec 26 11:47:29 CST 2018 now.

help
Option                               Description
------                               -----------
help                                 show help
quit                                 exit console mode
shutdown                             stop mpush server
restart                              restart mpush server
zk:<redis, cs ,gs>                   query zk node
count:<conn, online>                 count conn num or online user count
route:<uid>                          show user route info
push:<uid>, <msg>                    push test msg to client
conf:[key]                           show config info
monitor:[mxBean]                     show system monitor
profile:<1,0>                        enable/disable profile


count conn
0

push
unsupported option

route user01
user [user01] offline now.

profile 1
Profiler enabled

quit
have a good day!

Connection closed by foreign host.
➜  ~

```

mpush 后台输出日志：
```java
12:06:33.915 - [mp-admin-work-7-1] INFO  - com.mpush.core.handler.AdminHandler - receive admin command=help
12:06:45.343 - [mp-admin-work-7-1] INFO  - com.mpush.core.handler.AdminHandler - receive admin command=count conn
12:07:23.878 - [mp-admin-work-7-1] INFO  - com.mpush.core.handler.AdminHandler - receive admin command=push
12:08:14.383 - [mp-admin-work-7-1] INFO  - com.mpush.core.handler.AdminHandler - receive admin command=route user01
12:08:35.749 - [mp-admin-work-7-1] INFO  - com.mpush.core.handler.AdminHandler - receive admin command=profile 1
12:10:26.446 - [mp-admin-work-7-1] INFO  - com.mpush.core.handler.AdminHandler - receive admin command=quit

```
