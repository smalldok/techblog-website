---
title: docker run --restart and --rm
comments: true
tags:
  - docker
categories: 容器
abbrlink: 19968
date: 2018-07-09 10:12:00
---


# Docker容器的重启策略
Docker容器的重启策略是面向生产环境的一个启动策略，在开发过程中可以忽略该策略。  
Docker容器的重启都是由Docker守护进程完成的，因此与守护进程息息相关。  

Docker容器的重启策略如下：
* no，默认策略，在容器退出时不重启容器
* on-failure，在容器非正常退出时（退出状态非0），才会重启容器  
  on-failure:3，在容器非正常退出时重启容器，最多重启3次
* always，在容器退出时总是重启容器
* unless-stopped，在容器退出时总是重启容器，但是不考虑在Docker守护进程启动时就已经停止了的容器

# docker run的--restart选项
通过`--restart`选项，可以设置容器的重启策略，以决定在容器退出时Docker守护进程是否重启刚刚退出的容器。
`--restart`选项通常只用于detached模式的容器。
`--restart`选项不能与`--rm`选项同时使用。显然，`--restart`选项适用于detached模式的容器，而`--rm`选项适用于foreground模式的容器。

在`docker ps`查看容器时，对于使用了`--restart`选项的容器，其可能的状态只有`Up`或`Restarting`两种状态。

示例：

```
docker run -d --restart=always bba-208
docker run -d --restart=on-failure:10 bba-208
```

补充：

查看容器重启次数

```
docker inspect -f "{{ .RestartCount }}" bba-208
```
查看容器最后一次的启动时间
```
docker inspect -f "{{ .State.StartedAt }}" bba-208
```

# docker run的--rm选项
在Docker容器退出时，默认容器内部的文件系统仍然被保留，以方便调试并保留用户数据。
但是，对于foreground容器，由于其只是在开发调试过程中短期运行，其用户数据并无保留的必要，因而可以在容器启动时设置`--rm`选项，这样在容器退出时就能够自动清理容器内部的文件系统。示例如下：
```
docker run --rm bba-208
```

等价于
```
docker run --rm=true bba-208
```

显然，`--rm`选项不能与`-d`同时使用，即只能自动清理foreground容器，不能自动清理detached容器
注意，`--rm`选项也会清理容器的匿名data volumes。

所以，执行`docker run`命令带`--rm`命令选项，等价于在容器退出后，执行`docker rm -v`。
