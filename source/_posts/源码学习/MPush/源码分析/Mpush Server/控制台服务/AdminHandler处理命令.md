---
title: AdminHandler处理命令
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

1、注册命令处理器  
2、建立连接，打印欢迎画面  
3、接收命令，执行命令处理器  

# 注册命令处理器
初始化AdminHandler时，调用init()注册所有的命令处理器到内存中

```java
private final Map<String, OptionHandler> optionHandlers = new HashMap<>();

public AdminHandler(MPushServer mPushServer) {
    this.mPushServer = mPushServer;
    init();
}

public void init() {
    register("help", (ctx, args) ->
            "Option                               Description" + EOL +
                    "------                               -----------" + EOL +
                    "help                                 show help" + EOL +
                    "quit                                 exit console mode" + EOL +
                    "shutdown                             stop mpush server" + EOL +
                    "restart                              restart mpush server" + EOL +
                    "zk:<redis, cs ,gs>                   query zk node" + EOL +
                    "count:<conn, online>                 count conn num or online user count" + EOL +
                    "route:<uid>                          show user route info" + EOL +
                    "push:<uid>, <msg>                    push test msg to client" + EOL +
                    "conf:[key]                           show config info" + EOL +
                    "monitor:[mxBean]                     show system monitor" + EOL +
                    "profile:<1,0>                        enable/disable profile" + EOL
    );

    register("quit", (ctx, args) -> "have a good day!");

    register("shutdown", (ctx, args) -> {
        new Thread(() -> System.exit(0)).start();
        return "try close connect server...";
    });

    register("count", (ctx, args) -> {
        switch (args) {
            case "conn":
                return mPushServer.getConnectionServer().getConnectionManager().getConnNum();
            case "online": {
                return mPushServer.getRouterCenter().getUserEventConsumer().getUserManager().getOnlineUserNum();
            }

        }
        return "[" + args + "] unsupported, try help.";
    });

    register("route", (ctx, args) -> {
        if (Strings.isNullOrEmpty(args)) return "please input userId";
        Set<RemoteRouter> routers = mPushServer.getRouterCenter().getRemoteRouterManager().lookupAll(args);
        if (routers.isEmpty()) return "user [" + args + "] offline now.";
        return Jsons.toJson(routers);
    });

    register("conf", (ctx, args) -> {
        if (Strings.isNullOrEmpty(args)) {
            return CC.cfg.root().render(ConfigRenderOptions.concise().setFormatted(true));
        }
        if (CC.cfg.hasPath(args)) {
            return CC.cfg.getAnyRef(args).toString();
        }
        return "key [" + args + "] not find in config";
    });

    register("profile", (ctx, args) -> {
        if (args == null || "0".equals(args)) {
            Profiler.enable(false);
            return "Profiler disabled";
        } else {
            Profiler.enable(true);
            return "Profiler enabled";
        }
    });
}

private void register(String option, OptionHandler handler) {
    optionHandlers.put(option, handler);
}
public interface OptionHandler {
    Object handle(ChannelHandlerContext ctx, String args) throws Exception;
}

```

# 建立连接，打印欢迎画面
```java
@Override
public void channelActive(ChannelHandlerContext ctx) throws Exception {
    ctx.write("Welcome to MPush Console [" + ConfigTools.getLocalIp() + "]!" + EOL);
    ctx.write("since " + startTime + " has running " + startTime.until(LocalDateTime.now(), ChronoUnit.HOURS) + "(h)" + EOL + EOL);
    ctx.write("It is " + new Date() + " now." + EOL + EOL);
    ctx.flush();
}
```

```java
➜  ~ telnet 172.16.177.134 3002
Trying 172.16.177.134...
Connected to 172.16.177.134.
Escape character is '^]'.
Welcome to MPush Console [172.16.177.134]!
since 2018-12-26T11:47:00.087 has running 0(h)

It is Wed Dec 26 11:47:29 CST 2018 now.

```

# 接收命令，执行命令处理器
```java
//当读取到消息时，Netty触发channelRead()
@Override
protected void channelRead0(ChannelHandlerContext ctx, String request) throws Exception {
    String option = "help";
    String arg = null;
    String[] args = null;
    if (!Strings.isNullOrEmpty(request)) {
        String[] cmd_args = request.split(" ");
        option = cmd_args[0].trim().toLowerCase();
        if (cmd_args.length == 2) {
            arg = cmd_args[1];
        } else if (cmd_args.length > 2) {
            args = Arrays.copyOfRange(cmd_args, 1, cmd_args.length);
        }
    }
    try {
        //接收命令，并处理
        Object result = optionHandlers.getOrDefault(option, unsupported_handler).handle(ctx, arg);
        //将处理结果输出到控制台
        ChannelFuture future = ctx.writeAndFlush(result + EOL + EOL);
        //如果当前操作命令是quit，则监听CLOSE事件，做关闭连接处理
        if (option.equals("quit")) {
            future.addListener(ChannelFutureListener.CLOSE);
        }
    } catch (Throwable throwable) {
        ctx.writeAndFlush(throwable.getLocalizedMessage() + EOL + EOL);
        StringWriter writer = new StringWriter(1024);
        throwable.printStackTrace(new PrintWriter(writer));
        ctx.writeAndFlush(writer.toString());
    }
    LOGGER.info("receive admin command={}", request);
}

//在通道读取完成后会在这个方法里通知，对应可以做刷新操作 ctx.flush()
@Override
public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
    ctx.flush();
}

```
