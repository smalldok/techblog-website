---
title: 接收MPNS 广播PUSH消息
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

处理PUSH消息：
```java
public final class GatewayPushHandler extends BaseMessageHandler<GatewayPushMessage> {
    private final PushCenter pushCenter;
    public GatewayPushHandler(PushCenter pushCenter) {
        this.pushCenter = pushCenter;
    }
    @Override
    public GatewayPushMessage decode(Packet packet, Connection connection) {
        return new GatewayPushMessage(packet, connection);
    }
    @Override
    public void handle(GatewayPushMessage message) {
        pushCenter.push(message);
    }
}
```

消息处理，走的是 BroadcastPushTask 任务
```java
//PushCenter.java
@Override
public void push(IPushMessage message) {
    if (message.isBroadcast()) {
        FlowControl flowControl = (message.getTaskId() == null)
                ? new FastFlowControl(limit, max, duration)
                : new RedisFlowControl(message.getTaskId(), max);
        //添加到自定义的PushTaskTimer线程池中执行该任务
        addTask(new BroadcastPushTask(mPushServer, message, flowControl));
    } else {
        //添加到GatewayUDPConnector的netty work 线程池中执行该任务
        addTask(new SingleUserPushTask(mPushServer, message, globalFlowControl));
    }
}

```

自定义的PushTaskTimer线程池:
```java
//PushCenter.java

@Override
protected void doStart(Listener listener) throws Throwable {
    this.pushListener = PushListenerFactory.create();
    this.pushListener.init(mPushServer);
    if (CC.mp.net.udpGateway() || CC.mp.thread.pool.push_task > 0) {
        executor = new CustomJDKExecutor(mPushServer.getMonitor().getThreadPoolManager().getPushTaskTimer());
    } else {//实际情况使用EventLoo并没有更快，还有待测试
        executor = new NettyEventLoopExecutor();
    }
    MBeanRegistry.getInstance().register(new PushCenterBean(taskNum), null);
    ackTaskQueue.start();
    logger.info("push center start success");
    listener.onSuccess();
}

/**
 * UDP 模式使用自定义线程池
 */
private static class CustomJDKExecutor implements PushTaskExecutor {
    private final ScheduledExecutorService executorService;

    private CustomJDKExecutor(ScheduledExecutorService executorService) {
        this.executorService = executorService;
    }
    @Override
    public void shutdown() {
        executorService.shutdown();
    }
    @Override
    public void addTask(PushTask task) {
        executorService.execute(task);
    }
    @Override
    public void delayTask(long delay, PushTask task) {
        executorService.schedule(task, delay, TimeUnit.NANOSECONDS);
    }
}


```

线程池调度任务时，执行 run() 方法
```java
public final class BroadcastPushTask implements PushTask {

    private final long begin = System.currentTimeMillis();

    private final AtomicInteger finishTasks = new AtomicInteger(0);

    private final TimeLine timeLine = new TimeLine();

    private final Set<String> successUserIds = new HashSet<>(1024);

    private final FlowControl flowControl;

    private final IPushMessage message;

    private final Condition condition;

    private final MPushServer mPushServer;

    //使用Iterator, 记录任务遍历到的位置，因为有流控，一次任务可能会被分批发送，而且还有在推送过程中上/下线的用户
    private final Iterator<Map.Entry<String, Map<Integer, LocalRouter>>> iterator;

    public BroadcastPushTask(MPushServer mPushServer, IPushMessage message, FlowControl flowControl) {
        this.mPushServer = mPushServer;
        this.message = message;
        this.flowControl = flowControl;
        this.condition = message.getCondition();
        this.iterator = mPushServer.getRouterCenter().getLocalRouterManager().routers().entrySet().iterator();
        this.timeLine.begin("push-center-begin");
    }

    @Override
    public void run() {
        flowControl.reset();
        boolean done = broadcast();
        if (done) {//done 广播结束
            if (finishTasks.addAndGet(flowControl.total()) == 0) {
                report();
            }
        } else {//没有结束，就延时进行下次任务 TODO 考虑优先级问题
            mPushServer.getPushCenter().delayTask(flowControl.getDelay(), this);
        }
        flowControl.end(successUserIds.toArray(new String[successUserIds.size()]));
    }

    private boolean broadcast() {
        try {
            iterator.forEachRemaining(entry -> {

                String userId = entry.getKey();
                entry.getValue().forEach((clientType, router) -> {

                    Connection connection = router.getRouteValue();

                    if (checkCondition(condition, connection)) {//1.条件检测
                        if (connection.isConnected()) {
                            if (connection.getChannel().isWritable()) { //检测TCP缓冲区是否已满且写队列超过最高阀值
                                PushMessage
                                        .build(connection)
                                        .setContent(message.getContent())
                                        .send(future -> operationComplete(future, userId));
                                //4. 检测qps, 是否超过流控限制，如果超过则结束当前循环直接进入catch
                                if (!flowControl.checkQps()) {
                                    throw new OverFlowException(false);
                                }
                            }
                        } else { //2.如果链接失效，先删除本地失效的路由，再查下远程路由，看用户是否登陆到其他机器
                            Logs.PUSH.warn("[Broadcast] find router in local but conn disconnect, message={}, conn={}", message, connection);
                            //删除已经失效的本地路由
                            mPushServer.getRouterCenter().getLocalRouterManager().unRegister(userId, clientType);
                        }
                    }

                });

            });
        } catch (OverFlowException e) {
            //超出最大限制，或者遍历完毕，结束广播
            return e.isOverMaxLimit() || !iterator.hasNext();
        }
        return !iterator.hasNext();//遍历完毕, 广播结束
    }

    private void report() {
        Logs.PUSH.info("[Broadcast] task finished, cost={}, message={}", (System.currentTimeMillis() - begin), message);
        mPushServer.getPushCenter().getPushListener().onBroadcastComplete(message, timeLine.end().getTimePoints());//通知发送方，广播推送完毕
    }

    private boolean checkCondition(Condition condition, Connection connection) {
        if (condition == AwaysPassCondition.I) return true;
        SessionContext sessionContext = connection.getSessionContext();
        Map<String, Object> env = new HashMap<>();
        env.put("userId", sessionContext.userId);
        env.put("tags", sessionContext.tags);
        env.put("clientVersion", sessionContext.clientVersion);
        env.put("osName", sessionContext.osName);
        env.put("osVersion", sessionContext.osVersion);
        return condition.test(env);
    }

    //@Override
    private void operationComplete(ChannelFuture future, String userId) throws Exception {
        if (future.isSuccess()) {//推送成功
            successUserIds.add(userId);
            Logs.PUSH.info("[Broadcast] push message to client success, userId={}, message={}", message.getUserId(), message);
        } else {//推送失败
            Logs.PUSH.warn("[Broadcast] push message to client failure, userId={}, message={}, conn={}", message.getUserId(), message, future.channel());
        }
        if (finishTasks.decrementAndGet() == 0) {
            report();
        }
    }

    @Override
    public ScheduledExecutorService getExecutor() {
        return ((Message) message).getConnection().getChannel().eventLoop();
    }
}

```
