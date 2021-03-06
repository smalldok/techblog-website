---
title: CC-配置中心
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

Typesafe的Config库，纯Java写成、零外部依赖、代码精简、功能灵活、API友好。  
支持Java properties、JSON、JSON超集格式HOCON以及环境变量。
```java
public class Configure {
    private final Config config;
    public Configure(String confFileName) {
        config = ConfigFactory.load(confFileName);
    }
    public Configure() {
        config = ConfigFactory.load();
    }
    public String getString(String name) {
        return config.getString(name);
    }
}
```
ConfigFactory.load()会加载配置文件，默认加载classpath下的application.conf,application.json和application.properties文件。  
当然也可以调用ConfigFactory.load(confFileName)加载指定的配置文件。

配置内容即可以是层级关系，也可以用”.”号分隔写成一行:
```java
mp.log-level=${log.level}
mp.net.gateway-server-net=tcp // 网关服务使用的网络 udp/tcp
mp.net.alloc-server-port=9999
mp.net.alloc-server-protocol=http
mp.zk.server-address="127.0.0.1:2181"
mp.redis {// redis 集群配置
    password:""
    nodes:["127.0.0.1:6379"] //格式是ip:port
    cluster-model:single //single, cluster
}
// 或者
host{
  ip = 127.0.0.1
  port = 2282
}
// 或者
host.ip = 127.0.0.1
host.port = 2282

```
即json格式和properties格式。（貌似*.json只能是json格式，*.properties只能是properties格式，而*.conf可以是两者混合,而且配置文件只能是以上三种后缀名）

代码中如何使用：`String zk_address= CC.mp.zk.server-address;`

CC.java
```java
package com.mpush.tools.config;

import com.mpush.api.spi.net.DnsMapping;
import com.mpush.tools.config.data.RedisNode;
import com.typesafe.config.*;

import java.io.File;
import java.time.Duration;
import java.util.*;
import java.util.concurrent.TimeUnit;

import static java.util.stream.Collectors.toCollection;

/**
 * mpush 配置中心
 * Created by yxx on 2016/5/20.
 *
 * @author ohun@live.cn
 */
public interface CC {
    Config cfg = load();

    static Config load() {
        Config config = ConfigFactory.load();//扫描加载所有可用的配置文件
        // String custom_conf = System.getProperty("mp.conf");//加载自定义配置, 值来自jvm启动参数指定-Dmp.conf
        String custom_conf = "mp.conf";
        if (config.hasPath(custom_conf)) {
            File file = new File(config.getString(custom_conf));
            if (file.exists()) {
                Config custom = ConfigFactory.parseFile(file);
                config = custom.withFallback(config);
            }
        }
        return config;
    }

    interface mp {
        Config cfg = CC.cfg.getObject("mp").toConfig();
        String log_dir = cfg.getString("log-dir");
        String log_level = cfg.getString("log-level");
        String log_conf_path = cfg.getString("log-conf-path");

        interface core {
            Config cfg = mp.cfg.getObject("core").toConfig();

            int session_expired_time = (int) cfg.getDuration("session-expired-time").getSeconds();

            int max_heartbeat = (int) cfg.getDuration("max-heartbeat", TimeUnit.MILLISECONDS);

            int max_packet_size = (int) cfg.getMemorySize("max-packet-size").toBytes();

            int min_heartbeat = (int) cfg.getDuration("min-heartbeat", TimeUnit.MILLISECONDS);

            long compress_threshold = cfg.getBytes("compress-threshold");

            int max_hb_timeout_times = cfg.getInt("max-hb-timeout-times");

            String epoll_provider = cfg.getString("epoll-provider");

            static boolean useNettyEpoll() {
                if (!"netty".equals(CC.mp.core.epoll_provider)) return false;
                String name = CC.cfg.getString("os.name").toLowerCase(Locale.UK).trim();
                return name.startsWith("linux");//只在linux下使用netty提供的epoll库
            }
        }

        interface net {
            Config cfg = mp.cfg.getObject("net").toConfig();

            String local_ip = cfg.getString("local-ip");
            String public_ip = cfg.getString("public-ip");

            int connect_server_port = cfg.getInt("connect-server-port");
            String connect_server_bind_ip = cfg.getString("connect-server-bind-ip");
            String connect_server_register_ip = cfg.getString("connect-server-register-ip");
            Map<String, Object> connect_server_register_attr = cfg.getObject("connect-server-register-attr").unwrapped();


            int admin_server_port = cfg.getInt("admin-server-port");

            int gateway_server_port = cfg.getInt("gateway-server-port");
            String gateway_server_bind_ip = cfg.getString("gateway-server-bind-ip");
            String gateway_server_register_ip = cfg.getString("gateway-server-register-ip");
            String gateway_server_net = cfg.getString("gateway-server-net");
            String gateway_server_multicast = cfg.getString("gateway-server-multicast");
            String gateway_client_multicast = cfg.getString("gateway-client-multicast");
            int gateway_client_port = cfg.getInt("gateway-client-port");

            int ws_server_port = cfg.getInt("ws-server-port");
            String ws_path = cfg.getString("ws-path");
            int gateway_client_num = cfg.getInt("gateway-client-num");

            static boolean tcpGateway() {
                return "tcp".equals(gateway_server_net);
            }

            static boolean udpGateway() {
                return "udp".equals(gateway_server_net);
            }

            static boolean wsEnabled() {
                return ws_server_port > 0;
            }

            static boolean udtGateway() {
                return "udt".equals(gateway_server_net);
            }

            static boolean sctpGateway() {
                return "sctp".equals(gateway_server_net);
            }


            interface public_ip_mapping {

                Map<String, Object> mappings = net.cfg.getObject("public-host-mapping").unwrapped();

                static String getString(String localIp) {
                    return (String) mappings.get(localIp);
                }
            }

            interface snd_buf {
                Config cfg = net.cfg.getObject("snd_buf").toConfig();
                int connect_server = (int) cfg.getMemorySize("connect-server").toBytes();
                int gateway_server = (int) cfg.getMemorySize("gateway-server").toBytes();
                int gateway_client = (int) cfg.getMemorySize("gateway-client").toBytes();
            }

            interface rcv_buf {
                Config cfg = net.cfg.getObject("rcv_buf").toConfig();
                int connect_server = (int) cfg.getMemorySize("connect-server").toBytes();
                int gateway_server = (int) cfg.getMemorySize("gateway-server").toBytes();
                int gateway_client = (int) cfg.getMemorySize("gateway-client").toBytes();
            }

            interface write_buffer_water_mark {
                Config cfg = net.cfg.getObject("write-buffer-water-mark").toConfig();
                int connect_server_low = (int) cfg.getMemorySize("connect-server-low").toBytes();
                int connect_server_high = (int) cfg.getMemorySize("connect-server-high").toBytes();
                int gateway_server_low = (int) cfg.getMemorySize("gateway-server-low").toBytes();
                int gateway_server_high = (int) cfg.getMemorySize("gateway-server-high").toBytes();
            }

            interface traffic_shaping {
                Config cfg = net.cfg.getObject("traffic-shaping").toConfig();

                interface gateway_client {
                    Config cfg = traffic_shaping.cfg.getObject("gateway-client").toConfig();
                    boolean enabled = cfg.getBoolean("enabled");
                    long check_interval = cfg.getDuration("check-interval", TimeUnit.MILLISECONDS);
                    long write_global_limit = cfg.getBytes("write-global-limit");
                    long read_global_limit = cfg.getBytes("read-global-limit");
                    long write_channel_limit = cfg.getBytes("write-channel-limit");
                    long read_channel_limit = cfg.getBytes("read-channel-limit");
                }

                interface gateway_server {
                    Config cfg = traffic_shaping.cfg.getObject("gateway-server").toConfig();
                    boolean enabled = cfg.getBoolean("enabled");
                    long check_interval = cfg.getDuration("check-interval", TimeUnit.MILLISECONDS);
                    long write_global_limit = cfg.getBytes("write-global-limit");
                    long read_global_limit = cfg.getBytes("read-global-limit");
                    long write_channel_limit = cfg.getBytes("write-channel-limit");
                    long read_channel_limit = cfg.getBytes("read-channel-limit");
                }

                interface connect_server {
                    Config cfg = traffic_shaping.cfg.getObject("connect-server").toConfig();
                    boolean enabled = cfg.getBoolean("enabled");
                    long check_interval = cfg.getDuration("check-interval", TimeUnit.MILLISECONDS);
                    long write_global_limit = cfg.getBytes("write-global-limit");
                    long read_global_limit = cfg.getBytes("read-global-limit");
                    long write_channel_limit = cfg.getBytes("write-channel-limit");
                    long read_channel_limit = cfg.getBytes("read-channel-limit");
                }
            }
        }

        interface security {

            Config cfg = mp.cfg.getObject("security").toConfig();

            int aes_key_length = cfg.getInt("aes-key-length");

            String public_key = cfg.getString("public-key");

            String private_key = cfg.getString("private-key");

        }

        interface thread {

            Config cfg = mp.cfg.getObject("thread").toConfig();

            interface pool {

                Config cfg = thread.cfg.getObject("pool").toConfig();

                int conn_work = cfg.getInt("conn-work");
                int http_work = cfg.getInt("http-work");
                int push_task = cfg.getInt("push-task");
                int push_client = cfg.getInt("push-client");
                int ack_timer = cfg.getInt("ack-timer");
                int gateway_server_work = cfg.getInt("gateway-server-work");
                int gateway_client_work = cfg.getInt("gateway-client-work");

                interface event_bus {
                    Config cfg = pool.cfg.getObject("event-bus").toConfig();
                    int min = cfg.getInt("min");
                    int max = cfg.getInt("max");
                    int queue_size = cfg.getInt("queue-size");

                }

                interface mq {
                    Config cfg = pool.cfg.getObject("mq").toConfig();
                    int min = cfg.getInt("min");
                    int max = cfg.getInt("max");
                    int queue_size = cfg.getInt("queue-size");
                }
            }
        }

        interface zk {

            Config cfg = mp.cfg.getObject("zk").toConfig();

            int sessionTimeoutMs = (int) cfg.getDuration("sessionTimeoutMs", TimeUnit.MILLISECONDS);

            String watch_path = cfg.getString("watch-path");

            int connectionTimeoutMs = (int) cfg.getDuration("connectionTimeoutMs", TimeUnit.MILLISECONDS);

            String namespace = cfg.getString("namespace");

            String digest = cfg.getString("digest");

            String server_address = cfg.getString("server-address");

            interface retry {

                Config cfg = zk.cfg.getObject("retry").toConfig();

                int maxRetries = cfg.getInt("maxRetries");

                int baseSleepTimeMs = (int) cfg.getDuration("baseSleepTimeMs", TimeUnit.MILLISECONDS);

                int maxSleepMs = (int) cfg.getDuration("maxSleepMs", TimeUnit.MILLISECONDS);
            }
        }

        interface redis {
            Config cfg = mp.cfg.getObject("redis").toConfig();

            String password = cfg.getString("password");
            String clusterModel = cfg.getString("cluster-model");
            String sentinelMaster = cfg.getString("sentinel-master");

            List<RedisNode> nodes = cfg.getList("nodes")
                    .stream()//第一纬度数组
                    .map(v -> RedisNode.from(v.unwrapped().toString()))
                    .collect(toCollection(ArrayList::new));

            static boolean isCluster() {
                return "cluster".equals(clusterModel);
            }

            static boolean isSentinel() {
                return "sentinel".equals(clusterModel);
            }

            static <T> T getPoolConfig(Class<T> clazz) {
                return ConfigBeanImpl.createInternal(cfg.getObject("config").toConfig(), clazz);
            }
        }

        interface http {

            Config cfg = mp.cfg.getObject("http").toConfig();
            boolean proxy_enabled = cfg.getBoolean("proxy-enabled");
            int default_read_timeout = (int) cfg.getDuration("default-read-timeout", TimeUnit.MILLISECONDS);
            int max_conn_per_host = cfg.getInt("max-conn-per-host");


            long max_content_length = cfg.getBytes("max-content-length");

            Map<String, List<DnsMapping>> dns_mapping = loadMapping();

            static Map<String, List<DnsMapping>> loadMapping() {
                Map<String, List<DnsMapping>> map = new HashMap<>();
                cfg.getObject("dns-mapping").forEach((s, v) ->
                        map.put(s, ConfigList.class.cast(v)
                                .stream()
                                .map(cv -> DnsMapping.parse((String) cv.unwrapped()))
                                .collect(toCollection(ArrayList::new))
                        )
                );
                return map;
            }
        }

        interface push {

            Config cfg = mp.cfg.getObject("push").toConfig();

            interface flow_control {

                Config cfg = push.cfg.getObject("flow-control").toConfig();

                interface global {
                    Config cfg = flow_control.cfg.getObject("global").toConfig();
                    int limit = cfg.getNumber("limit").intValue();
                    int max = cfg.getInt("max");
                    int duration = (int) cfg.getDuration("duration").toMillis();
                }

                interface broadcast {
                    Config cfg = flow_control.cfg.getObject("broadcast").toConfig();
                    int limit = cfg.getInt("limit");
                    int max = cfg.getInt("max");
                    int duration = (int) cfg.getDuration("duration").toMillis();
                }
            }
        }

        interface monitor {
            Config cfg = mp.cfg.getObject("monitor").toConfig();
            String dump_dir = cfg.getString("dump-dir");
            boolean dump_stack = cfg.getBoolean("dump-stack");
            boolean print_log = cfg.getBoolean("print-log");
            Duration dump_period = cfg.getDuration("dump-period");
            boolean profile_enabled = cfg.getBoolean("profile-enabled");
            Duration profile_slowly_duration = cfg.getDuration("profile-slowly-duration");
        }
    }
}

```


<br>
 功能组件文章目录:

* **[CC-配置中心](../CC-配置中心)**
* [EventBus-事件总线](../EventBus-事件总线)
* [FlowControl-流控](../FlowControl-流控)
* [JVMUtil](../JVMUtil)
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
