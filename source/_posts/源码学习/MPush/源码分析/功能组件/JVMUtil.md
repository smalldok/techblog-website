---
title: JVMUtil
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

利用JMX的MBEAN,输出线程栈、堆栈到本地；

# 使用时机1（线程池发生reject，打印线程栈）
```java
package com.mpush.tools.thread.pool;

import com.mpush.tools.Utils;
import com.mpush.tools.common.JVMUtil;
import com.mpush.tools.config.CC;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.concurrent.RejectedExecutionException;
import java.util.concurrent.RejectedExecutionHandler;
import java.util.concurrent.ThreadPoolExecutor;

import static com.mpush.tools.thread.pool.ThreadPoolConfig.REJECTED_POLICY_ABORT;
import static com.mpush.tools.thread.pool.ThreadPoolConfig.REJECTED_POLICY_CALLER_RUNS;

public final class DumpThreadRejectedHandler implements RejectedExecutionHandler {

    private final static Logger LOGGER = LoggerFactory.getLogger(DumpThreadRejectedHandler.class);

    private volatile boolean dumping = false;

    private static final String DUMP_DIR = CC.mp.monitor.dump_dir;

    private final ThreadPoolConfig poolConfig;

    private final int rejectedPolicy;

    public DumpThreadRejectedHandler(ThreadPoolConfig poolConfig) {
        this.poolConfig = poolConfig;
        this.rejectedPolicy = poolConfig.getRejectedPolicy();
    }

    @Override
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        LOGGER.warn("one task rejected, poolConfig={}, poolInfo={}", poolConfig, Utils.getPoolInfo(e));
        if (!dumping) {
            dumping = true;
            dumpJVMInfo();
        }

        if (rejectedPolicy == REJECTED_POLICY_ABORT) {
            throw new RejectedExecutionException("one task rejected, pool=" + poolConfig.getName());
        } else if (rejectedPolicy == REJECTED_POLICY_CALLER_RUNS) {
            if (!e.isShutdown()) {
                r.run();
            }
        }
    }

    private void dumpJVMInfo() {
        LOGGER.info("start dump jvm info");
        JVMUtil.dumpJstack(DUMP_DIR + "/" + poolConfig.getName());
        LOGGER.info("end dump jvm info");
    }
}
```

# 使用时机2（MPushServer监控，打印线程栈、堆栈）
```java
public class MonitorService extends BaseService implements Monitor, Runnable {

    private static final int FIRST_DUMP_JSTACK_LOAD_AVG = 2,
            SECOND_DUMP_JSTACK_LOAD_AVG = 4,
            THIRD_DUMP_JSTACK_LOAD_AVG = 6,
            FIRST_DUMP_JMAP_LOAD_AVG = 4;

    private static final String dumpLogDir = CC.mp.monitor.dump_dir;
    private static final boolean dumpEnabled = CC.mp.monitor.dump_stack;
    private static final boolean printLog = CC.mp.monitor.print_log;
    private static final long dumpPeriod = CC.mp.monitor.dump_period.getSeconds();

    private volatile boolean dumpFirstJstack = false;
    private volatile boolean dumpSecondJstack = false;
    private volatile boolean dumpThirdJstack = false;
    private volatile boolean dumpJmap = false;

    private final ResultCollector collector;

    private final ThreadPoolManager threadPoolManager;

    public MonitorService() {
        threadPoolManager = new ThreadPoolManager();
        collector = new ResultCollector(threadPoolManager);
    }

    private Thread thread;

    @Override
    public void run() {
        while (isRunning()) {
            MonitorResult result = collector.collect();

            if (printLog) {
                Logs.MONITOR.info(result.toJson());
            }

            if (dumpEnabled) {
                dump();
            }

            try {
                TimeUnit.SECONDS.sleep(dumpPeriod);
            } catch (InterruptedException e) {
                if (isRunning()) stop();
            }
        }
    }

    @Override
    protected void doStart(Listener listener) throws Throwable {
        if (printLog || dumpEnabled) {
            thread = Utils.newThread(ThreadNames.T_MONITOR, this);
            thread.setDaemon(true);
            thread.start();
        }
        listener.onSuccess();
    }

    @Override
    protected void doStop(Listener listener) throws Throwable {
        if (thread != null && thread.isAlive()) thread.interrupt();
        listener.onSuccess();
    }

    private void dump() {
        double load = collector.getJvmInfo().load();
        if (load > FIRST_DUMP_JSTACK_LOAD_AVG) {
            if (!dumpFirstJstack) {
                dumpFirstJstack = true;
                JVMUtil.dumpJstack(dumpLogDir);
            }
        }

        if (load > SECOND_DUMP_JSTACK_LOAD_AVG) {
            if (!dumpSecondJstack) {
                dumpSecondJstack = true;
                JVMUtil.dumpJmap(dumpLogDir);
            }
        }

        if (load > THIRD_DUMP_JSTACK_LOAD_AVG) {
            if (!dumpThirdJstack) {
                dumpThirdJstack = true;
                JVMUtil.dumpJmap(dumpLogDir);
            }
        }

        if (load > FIRST_DUMP_JMAP_LOAD_AVG) {
            if (!dumpJmap) {
                dumpJmap = true;
                JVMUtil.dumpJmap(dumpLogDir);
            }
        }
    }


    @Override
    public void monitor(String name, Thread thread) {

    }

    @Override
    public void monitor(String name, Executor executor) {
        threadPoolManager.register(name, executor);
    }

    public ThreadPoolManager getThreadPoolManager() {
        return threadPoolManager;
    }
}

```

源码：
```java
/*
 * (C) Copyright 2015-2016 the original author or authors.
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 *
 * Contributors:
 *   ohun@live.cn (夜色)
 */

package com.mpush.tools.common;

import com.mpush.tools.Utils;
import com.sun.management.HotSpotDiagnosticMXBean;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import javax.management.MBeanServer;
import javax.management.ObjectName;
import java.io.File;
import java.io.FileOutputStream;
import java.io.OutputStream;
import java.io.PrintStream;
import java.lang.management.*;
import java.security.AccessController;
import java.security.PrivilegedExceptionAction;
import java.util.Iterator;
import java.util.Map;
import java.util.Set;

public class JVMUtil {
    private static final String HOT_SPOT_BEAN_NAME = "com.sun.management:type=HotSpotDiagnostic";

    private static final Logger LOGGER = LoggerFactory.getLogger(JVMUtil.class);

    private static HotSpotDiagnosticMXBean hotSpotMXBean;
    private static ThreadMXBean threadMXBean = ManagementFactory.getThreadMXBean();

    public static void jstack(OutputStream stream) throws Exception {
        PrintStream out = new PrintStream(stream);
        boolean cpuTimeEnabled = threadMXBean.isThreadCpuTimeSupported() && threadMXBean.isThreadCpuTimeEnabled();
        Map<Thread, StackTraceElement[]> map = Thread.getAllStackTraces();

        for (Map.Entry<Thread, StackTraceElement[]> entry : map.entrySet()) {
            Thread t = entry.getKey();
            StackTraceElement[] elements = entry.getValue();

            ThreadInfo tt = threadMXBean.getThreadInfo(t.getId());
            long tid = t.getId();
            Thread.State state = t.getState();
            long cpuTimeMillis = cpuTimeEnabled ? threadMXBean.getThreadCpuTime(tid) / 1000000 : -1;
            long userTimeMillis = cpuTimeEnabled ? threadMXBean.getThreadUserTime(tid) / 1000000 : -1;

            out.printf("%s id=%d state=%s deamon=%s priority=%s cpu[total=%sms,user=%sms]", t.getName(),
                    tid, t.getState(), t.isDaemon(), t.getPriority(), cpuTimeMillis, userTimeMillis);
            final LockInfo lock = tt.getLockInfo();
            if (lock != null && state != Thread.State.BLOCKED) {
                out.printf("%n    - waiting on <0x%08x> (a %s)", lock.getIdentityHashCode(), lock.getClassName());
                out.printf("%n    - locked <0x%08x> (a %s)", lock.getIdentityHashCode(), lock.getClassName());
            } else if (lock != null && state == Thread.State.BLOCKED) {
                out.printf("%n    - waiting to lock <0x%08x> (a %s)", lock.getIdentityHashCode(),
                        lock.getClassName());
            }

            if (tt.isSuspended()) {
                out.print(" (suspended)");
            }

            if (tt.isInNative()) {
                out.print(" (running in native)");
            }

            out.println();
            if (tt.getLockOwnerName() != null) {
                out.printf("     owned by %s id=%d%n", tt.getLockOwnerName(), tt.getLockOwnerId());
            }

            final MonitorInfo[] monitors = tt.getLockedMonitors();

            for (int i = 0; i < elements.length; i++) {
                final StackTraceElement element = elements[i];
                out.printf("    at %s%n", element);
                for (int j = 1; j < monitors.length; j++) {
                    final MonitorInfo monitor = monitors[j];
                    if (monitor.getLockedStackDepth() == i) {
                        out.printf("      - locked %s%n", monitor);
                    }
                }
            }

            out.println();

            final LockInfo[] locks = tt.getLockedSynchronizers();
            if (locks.length > 0) {
                out.printf("    Locked synchronizers: count = %d%n", locks.length);
                for (LockInfo l : locks) {
                    out.printf("      - %s%n", l);
                }
                out.println();
            }
        }
    }

    public static void dumpJstack(final String jvmPath) {
        Utils.newThread("dump-jstack-t", (() -> {
            File path = new File(jvmPath);
            if (path.exists() || path.mkdirs()) {
                File file = new File(path, System.currentTimeMillis() + "-jstack.log");
                try (FileOutputStream out = new FileOutputStream(file)) {
                    JVMUtil.jstack(out);
                } catch (Throwable t) {
                    LOGGER.error("Dump JVM cache Error!", t);
                }
            }
        })).start();
    }

    private static HotSpotDiagnosticMXBean getHotSpotMXBean() {
        try {
            return AccessController.doPrivileged(new PrivilegedExceptionAction<HotSpotDiagnosticMXBean>() {
                public HotSpotDiagnosticMXBean run() throws Exception {
                    MBeanServer server = ManagementFactory.getPlatformMBeanServer();
                    Set<ObjectName> s = server.queryNames(new ObjectName(HOT_SPOT_BEAN_NAME), null);
                    Iterator<ObjectName> itr = s.iterator();
                    if (itr.hasNext()) {
                        ObjectName name = itr.next();
                        HotSpotDiagnosticMXBean bean = ManagementFactory.newPlatformMXBeanProxy(server,
                                name.toString(), HotSpotDiagnosticMXBean.class);
                        return bean;
                    } else {
                        return null;
                    }
                }
            });
        } catch (Exception e) {
            LOGGER.error("getHotSpotMXBean Error!", e);
            return null;
        }
    }

    private static void initHotSpotMBean() throws Exception {
        if (hotSpotMXBean == null) {
            synchronized (JVMUtil.class) {
                if (hotSpotMXBean == null) {
                    hotSpotMXBean = getHotSpotMXBean();
                }
            }
        }
    }

    public static void jMap(String fileName, boolean live) {
        File f = new File(fileName, System.currentTimeMillis() + "-jmap.log");
        String currentFileName = f.getPath();
        try {
            initHotSpotMBean();
            if (f.exists()) {
                f.delete();
            }
            hotSpotMXBean.dumpHeap(currentFileName, live);
        } catch (Exception e) {
            LOGGER.error("dumpHeap Error!" + currentFileName, e);
        }
    }

    public static void dumpJmap(final String jvmPath) {
        Utils.newThread("dump-jmap-t", () -> jMap(jvmPath, false)).start();
    }
}
```

<br>
 功能组件文章目录:

* [CC-配置中心](../CC-配置中心)
* [EventBus-事件总线](../EventBus-事件总线)
* [FlowControl-流控](../FlowControl-流控)
* **[JVMUtil](../JVMUtil)**
* [Logs](../Logs)
* [Profiler-统计方法或者线程执行时间](../Profiler-统计方法或者线程执行时间)
* [Profiler入门](../Profiler入门)
* [SPI机制](../SPI机制)
* [TimeLine-时间线](../TimeLine-时间线)
* [服务启动监听](../服务启动监听)
* [监控](../监控)
* [通信模型与超时控制](../通信模型与超时控制)
* [线程池](../线程池)
* [状态判断-位运算](../状态判断-位运算)
