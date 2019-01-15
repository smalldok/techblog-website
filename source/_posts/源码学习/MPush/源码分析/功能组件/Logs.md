---
title: Logs
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

# 初始化
Logs.init();
* MPush启动时；
* 客户端连接时；

# 使用
```java
Logs.Console.warn("你正在使用的CacheManager只能用于源码测试，生产环境请使用redis 3.x.");
```

# 输出
```java
10:06:48.366 -[console]- 你正在使用的CacheManager只能用于源码测试，生产环境请使用redis 3.x.
```

# 工具类
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

package com.mpush.tools.log;

import com.mpush.tools.config.CC;
import com.typesafe.config.ConfigRenderOptions;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;


/**
 * Created by ohun on 2016/5/16.
 *
 * @author ohun@live.cn
 */
public interface Logs {
    boolean logInit = init();

    static boolean init() {
        if (logInit) return true;
        System.setProperty("log.home", CC.mp.log_dir);
        System.setProperty("log.root.level", CC.mp.log_level);
        System.setProperty("logback.configurationFile", CC.mp.log_conf_path);
        LoggerFactory
                .getLogger("console")
                .info(CC.mp.cfg.root().render(ConfigRenderOptions.concise().setFormatted(true)));
        return true;
    }

    Logger Console = LoggerFactory.getLogger("console"),

    CONN = LoggerFactory.getLogger("mpush.conn.log"),

    MONITOR = LoggerFactory.getLogger("mpush.monitor.log"),

    PUSH = LoggerFactory.getLogger("mpush.push.log"),

    HB = LoggerFactory.getLogger("mpush.heartbeat.log"),

    CACHE = LoggerFactory.getLogger("mpush.cache.log"),

    RSD = LoggerFactory.getLogger("mpush.srd.log"),

    HTTP = LoggerFactory.getLogger("mpush.http.log"),

    PROFILE = LoggerFactory.getLogger("mpush.profile.log");
}
```




<br>
 功能组件文章目录:

* [CC-配置中心](../CC-配置中心)
* [EventBus-事件总线](../EventBus-事件总线)
* [FlowControl-流控](../FlowControl-流控)
* [JVMUtil](../JVMUtil)
* **[Logs](../Logs)**
* [Profiler-统计方法或者线程执行时间](../Profiler-统计方法或者线程执行时间)
* [Profiler入门](../Profiler入门)
* [SPI机制](../SPI机制)
* [TimeLine-时间线](../TimeLine-时间线)
* [服务启动监听](../服务启动监听)
* [监控](../监控)
* [通信模型与超时控制](../通信模型与超时控制)
* [线程池](../线程池)
* [状态判断-位运算](../状态判断-位运算)
