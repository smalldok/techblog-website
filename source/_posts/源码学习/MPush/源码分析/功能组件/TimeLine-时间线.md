---
title: TimeLine-时间线
comments: true
tags:
  - 源码
  - 消息推送
categories:
  - 源码学习
  - MPush
  - 源码分析
  - 功能组件
abbrlink: 20000
date: 2016-03-01 21:00:00
---

TimeLine可以跟踪调用链路点，记录经过的各个环节点，从而知道这条消息经过了哪几个组件；

使用：
```java
public final class SingleUserPushTask implements PushTask, ChannelFutureListener {
    //初始化
    private final TimeLine timeLine = new TimeLine();
    public SingleUserPushTask(MPushServer mPushServer, IPushMessage message, FlowControl flowControl) {
        .....
        //标记开始
        this.timeLine.begin("push-center-begin");
    }
    ....
    mPushServer.getPushCenter().getPushListener().onTimeout(message, timeLine.timeoutEnd().getTimePoints());
    mPushServer.getPushCenter().getPushListener().onSuccess(message, timeLine.successEnd().getTimePoints());
    mPushServer.getPushCenter().getPushListener().onFailure(message, timeLine.failureEnd().getTimePoints());
    mPushServer.getPushCenter().getPushListener().onRedirect(message, timeLine.end("redirect-end").getTimePoints());
    mPushServer.getPushCenter().getPushListener().onOffline(message, timeLine.end("offline-end").getTimePoints());

    /**
     * 添加ACK任务到队列, 等待客户端响应
     * @param messageId 下发到客户端待ack的消息的sessionId
     */
    private void addAckTask(int messageId) {
        timeLine.addTimePoint("waiting-ack");
        //因为要进队列，可以提前释放一些比较占用内存的字段，便于垃圾回收
        message.finalized();
        AckTask task = AckTask
                .from(messageId)
                .setCallback(new PushAckCallback(message, timeLine, mPushServer.getPushCenter()));
        mPushServer.getPushCenter().getAckTaskQueue().add(task, message.getTimeoutMills() - (int) (System.currentTimeMillis() - start));
    }
}

```

```java
public final class TimeLine {
    private final TimePoint root = new TimePoint("root");
    private final String name;
    private int pointCount;
    private TimePoint current = root;

    public TimeLine() {
        name = "TimeLine";
    }

    public TimeLine(String name) {
        this.name = name;
    }

    public void begin(String name) {
        addTimePoint(name);
    }

    public void begin() {
        addTimePoint("begin");
    }

    public void addTimePoint(String name) {
        current = current.next = new TimePoint(name);
        pointCount++;
    }

    public void addTimePoints(Object[] points) {
        if (points != null) {
            for (int i = 0; i < points.length; i++) {
                current = current.next = new TimePoint((String) points[i], ((Number) points[++i]).longValue());
                pointCount++;
            }
        }
    }

    public TimeLine end(String name) {
        addTimePoint(name);
        return this;
    }

    public TimeLine end() {
        addTimePoint("end");
        return this;
    }

    public TimeLine successEnd() {
        addTimePoint("success-end");
        return this;
    }

    public TimeLine failureEnd() {
        addTimePoint("failure-end");
        return this;
    }

    public TimeLine timeoutEnd() {
        addTimePoint("timeout-end");
        return this;
    }

    public void clean() {
        root.next = null;
    }

    @Override
    public String toString() {
        StringBuilder sb = new StringBuilder(name);
        if (root.next != null) {
            sb.append('[').append(current.time - root.next.time).append(']').append("(ms)");
        }
        sb.append('{');
        TimePoint next = root;
        while ((next = next.next) != null) {
            sb.append(next.toString());
        }
        sb.append('}');
        return sb.toString();
    }

    public Object[] getTimePoints() {
        Object[] arrays = new Object[2 * pointCount];
        int i = 0;
        TimePoint next = root;
        while ((next = next.next) != null) {
            arrays[i++] = next.name;
            arrays[i++] = next.time;
        }
        return arrays;
    }

    private static class TimePoint {
        private final String name;
        private final long time;
        private transient TimePoint next;

        public TimePoint(String name) {
            this.name = name;
            this.time = System.currentTimeMillis();
        }

        public TimePoint(String name, long time) {
            this.name = name;
            this.time = time;
        }

        public void setNext(TimePoint next) {
            this.next = next;
        }

        @Override
        public String toString() {
            if (next == null) return name;
            return name + " --（" + (next.time - time) + "ms) --> ";
        }
    }
}

```



<br>
 功能组件文章目录:

* [CC-配置中心](../CC-配置中心)
* [EventBus-事件总线](../EventBus-事件总线)
* [FlowControl-流控](../FlowControl-流控)**
* [JVMUtil](../JVMUtil)
* [Logs](../Logs)
* [Profiler-统计方法或者线程执行时间](../Profiler-统计方法或者线程执行时间)
* [Profiler入门](../Profiler入门)
* [SPI机制](../SPI机制)**
* **[TimeLine-时间线](../TimeLine-时间线)**
* [服务启动监听](../服务启动监听)
* [监控](../监控)
* [通信模型与超时控制](../通信模型与超时控制)
* [线程池](../线程池)
* [状态判断-位运算](../状态判断-位运算)
