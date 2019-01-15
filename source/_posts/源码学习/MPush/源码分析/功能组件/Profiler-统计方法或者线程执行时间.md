---
title: Profiler-统计方法或者线程执行时间
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

# 测试
实现类： Profiler.java  
这两个类中打断点    
* AdminHandler.java  
* ServerChannelHandler.java  

测试Profiler
```java
public class ProfilerTest {
    //10ms
    long profile_slowly_limit = CC.mp.monitor.profile_slowly_duration.toMillis();
    @Test
    public void testProfiler() throws Exception {
        byte cmd = 1;
        try {
            Profiler.enable(true);
            Profiler.start("time cost on [channel read]: %s", "111111");

            Profiler.enter("time cost on [A]");
            Thread.sleep(300);
            Profiler.release();

            Profiler.enter("time cost on [B]");
            Thread.sleep(500);
            Profiler.release();

            Profiler.enter("time cost on [C]");
            Thread.sleep(200);
            Profiler.release();

            Thread.sleep(400);
        }finally {
            Profiler.release();
            if (Profiler.getDuration() > profile_slowly_limit) {
                System.out.println(Profiler.dump());
            }
            Profiler.reset();
        }
    }
}
```

输出：
```java
0 [1,400ms (400ms), 100%] - time cost on [channel read]:
+---0 [300ms, 21%, 21%] - time cost on [A]
+---300 [500ms, 36%, 36%] - time cost on [B]
`---800 [200ms, 14%, 14%] - time cost on [C]
```

# Profiler入门

[Profiler入门](../Profiler入门)

<br>
 功能组件文章目录:

* [CC-配置中心](../CC-配置中心)
* [EventBus-事件总线](../EventBus-事件总线)
* [FlowControl-流控](../FlowControl-流控)
* [JVMUtil](../JVMUtil)
* [Logs](../Logs)
* **[Profiler-统计方法或者线程执行时间](../Profiler-统计方法或者线程执行时间)**
* [Profiler入门](../Profiler入门)
* [SPI机制](../SPI机制)
* [TimeLine-时间线](../TimeLine-时间线)
* [服务启动监听](../服务启动监听)
* [监控](../监控)
* [通信模型与超时控制](../通信模型与超时控制)
* [线程池](../线程池)
* [状态判断-位运算](../状态判断-位运算)
