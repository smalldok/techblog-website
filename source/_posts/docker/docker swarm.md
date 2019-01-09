---
title: docker swarm mode
comments: true
date: 2018-07-6 16:12:00
tags:
  - docker  
categories: 容器
---

# 准备环境
* `三台linux服务器或者linux容器,并且安装docker环境`

这里我采用容器, 需docker run docker,也就是docker上运行centos容器，centos容器中再运行docker环境;  
构建容器环境，参考：https://github.com/smalldok/docker-based-tools/tree/master/dind_centos

本地构建dind_centos镜像之后,分别创建如下三个容器（对应好Ip和hostname）

| role | ip | hostname | desc |
|--------|--------|--------|--------|
| manager node | 172.17.0.2 | docker-0 |-|
| work node | 172.17.0.3 | docker-1 |-|
| work node | 172.17.0.4 | docker-2 |-|

* `主机间开放端口`  
在各个节点进行通信的时候,必须开放相关的防火墙策略,其中包括：  
tcp port 2377为集群管理通信  
tcp and udp port 7946为节点通信  
udp port 4789为网络间流量  

分别在各主机上执行如下命令：

```shell
# 开放防火墙端口
firewall-cmd --permanent --add-port=2377/tcp
firewall-cmd --permanent --add-port=7946/tcp
firewall-cmd --permanent --add-port=7946/udp
firewall-cmd --permanent --add-port=4789/udp
firewall-cmd --reload # 永久打开端口需要reload一下
firewall-cmd --list-all # 查看防火墙，添加的端口也可以看到
```

centos7镜像需要安装的工具

```shell
# ifconfig命令支持
yum install net-tools
# 安装firewalld
yum install firewalld firewall-config
systemctl start  firewalld # 启动
systemctl status firewalld # 或者 firewall-cmd --state 查看状态
```

# 创建swarm集群
`初始化集群，节点之间相互通信的IP地址为172.17.0.2，默认端口2377`

```shell
[root@docker-0 swarm]# docker swarm init --advertise-addr 172.17.0.2
Swarm initialized: current node (dlr9h76ov43r7239sq3iksjyw) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-3d7ynvjvs9kqdqaut2sir2ky42uoiv54fmwef5s7odei33a031-3rcwr1styfzw8f8j3k9mvbz9c 172.17.0.2:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

`swarm集群模式是否激活`

```shell
[root@docker-0 swarm]# docker info|grep -i swarm
Swarm: active
```

`默认监听两个TCP端口`

```shell
# 2377：集群的管理端口、7946：节点之间的通信端口
[root@docker-0 swarm]# netstat -tnlp|grep docker
tcp6       0      0 :::2377                 :::*                    LISTEN      78/dockerd      
tcp6       0      0 :::7946                 :::*                    LISTEN      78/dockerd 
```

`查看网络`

```shell
# 默认会创建一个overlay的网络ingress、一个bridge网络docker_gwbridge
[root@docker-0 swarm]# docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
301fd3e9c939        bridge              bridge              local
5ab7c9aec44f        docker_gwbridge     bridge              local
9faeade5067d        host                host                local
k0m6xm3c1pgr        ingress             overlay             swarm
74eb81b4e335        none                null                local
```

`查看集群中的节点,当有多个manager节点的时候,是通过raft协议来选取主节点,也就是leader节点`

```shell
[root@docker-0 swarm]# docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
dlr9h76ov43r7239sq3iksjyw *   docker-0            Ready               Active              Leader              18.03.1-ce
```

`swarm配置文件`

```shell
# swarm配置文件都在/var/lib/docker/swarm目录中,有相关的证书和manager的配置文件,使用的是raft协议
[root@docker-0 swarm]# ls -l
total 20
drwxr-xr-x 2 root root 4096 Jun 26 03:23 certificates  # 使用tls来进行安全通信
-rw------- 1 root root  145 Jun 26 03:23 docker-state.json # 用来记录通信的地址和端口,也会记录本地的地址和端口
drwx------ 4 root root 4096 Jun 26 03:23 raft # raft协议
-rw------- 1 root root   66 Jun 26 03:23 state.json  # manager的ip和端口
drwxr-xr-x 2 root root 4096 Jun 26 03:23 worker  # 记录工作节点下发的任务信息
```

`登陆docker-1, 将docker-1节点加入集群`

```shell
[root@docker-1 /]# docker swarm join --token SWMTKN-1-3d7ynvjvs9kqdqaut2sir2ky42uoiv54fmwef5s7odei33a031-3rcwr1styfzw8f8j3k9mvbz9c 172.17.0.2:2377
This node joined a swarm as a worker.
```

`登陆docker-2, 将docker-2节点加入集群`

```shell
[root@docker-2 /]# docker swarm join --token SWMTKN-1-3d7ynvjvs9kqdqaut2sir2ky42uoiv54fmwef5s7odei33a031-3rcwr1styfzw8f8j3k9mvbz9c 172.17.0.2:2377
This node joined a swarm as a worker.
```

`当忘记了加入集群的token,可以使用如下命令找到,然后在node节点上直接执行,就可以加入worker节点或者manager节点`

```shell
[root@docker-0 swarm]# docker swarm join-token worker
To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-3d7ynvjvs9kqdqaut2sir2ky42uoiv54fmwef5s7odei33a031-3rcwr1styfzw8f8j3k9mvbz9c 172.17.0.2:2377
```

`查看集群节点信息`

```shell
[root@docker-0 swarm]# docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
dlr9h76ov43r7239sq3iksjyw *   docker-0            Ready               Active              Leader              18.03.1-ce
wscczkpfb15c5knm6cr6lj712     docker-1            Ready               Active                                  18.03.1-ce
pgpnatt19yjti3eqy0gc3ebwg     docker-2            Ready               Active                                  18.03.1-ce
```

`节点之间的角色可以随时进行转换(使用update进行更新)`

```shell
[root@docker-0 swarm]# docker node update --role manager docker-1
docker-1
[root@docker-0 swarm]# docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
dlr9h76ov43r7239sq3iksjyw *   docker-0            Ready               Active              Leader              18.03.1-ce
wscczkpfb15c5knm6cr6lj712     docker-1            Ready               Active              Reachable           18.03.1-ce
pgpnatt19yjti3eqy0gc3ebwg     docker-2            Ready               Active                                  18.03.1-ce
[root@docker-0 swarm]# docker node update --role worker docker-1
docker-1
[root@docker-0 swarm]# docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
dlr9h76ov43r7239sq3iksjyw *   docker-0            Ready               Active              Leader              18.03.1-ce
wscczkpfb15c5knm6cr6lj712     docker-1            Ready               Active                                  18.03.1-ce
pgpnatt19yjti3eqy0gc3ebwg     docker-2            Ready               Active                                  18.03.1-ce
```

# 运行服务
service是一组task集合，包括多个task，一个task为一个容器;  
例如运行nginx服务,从而拆解为几个nginx的容器在各个节点上进行运行；

`创建名称为web的服务，基于nginx镜像`

```shell
[root@docker-0 swarm]# docker service create --name web nginx
plwzhwijh15j6tpg7yz7wshdp
overall progress: 1 out of 1 tasks
1/1: running
verify: Service converged
```

`创建名称为frontweb的服务，基于nginx镜像，模式为global`

```shell
[root@docker-0 swarm]# docker service create --name frontweb --mode global nginx
p2geazk5hdckzzb9rdd1m5n9l
overall progress: 3 out of 3 tasks
dlr9h76ov43r: running
wscczkpfb15c: running
pgpnatt19yjt: running
verify: Service converged
```

`查看运行的服务`

```shell
[root@docker-0 swarm]# docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE               PORTS
p2geazk5hdck        frontweb            global              3/3                 nginx:latest    
plwzhwijh15j        web                 replicated          1/1                 nginx:latest 
```

`查看web运行信息,默认情况下manager节点也可以运行容器`

```shell
[root@docker-0 swarm]# docker service ps web
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
qjrl2qg04a3b        web.1               nginx:latest        docker-0            Running             Running 15 minutes ago
```

`查看frontweb运行信息,默认情况下manager节点也可以运行容器`

```shell
[root@docker-0 swarm]# docker service ps frontweb
ID                  NAME                                 IMAGE               NODE                DESIRED STATE       CURRENT STATE           ERROR               PORTS
n7r3e85ukc2m        frontweb.pgpnatt19yjti3eqy0gc3ebwg   nginx:latest        docker-2            Running             Running 5 minutes ago
paf09ihxyr5q        frontweb.wscczkpfb15c5knm6cr6lj712   nginx:latest        docker-1            Running             Running 5 minutes ago
o7yrjm0jbll1        frontweb.dlr9h76ov43r7239sq3iksjyw   nginx:latest        docker-0            Running             Running 6 minutes ago
```

**创建服务时，服务的几种状态**  
prepared 表示准备,主要是从仓库拉取镜像；  
starting 启动容器,进行验证容器状态；  
running 正常运行  

**服务模式**  
replicated 表示副本，默认使用的mode，并且默认情况只会创建一个副本，主要使用目的是为了高可用;
global 必须在每个节点上运行一个task(也就是容器),可以看到上面创建frontweb服务时，使用global创建了三个容器;

查看frontweb服务详细信息

```shell
[root@docker-0 swarm]# docker service inspect frontweb --pretty

ID:             p2geazk5hdckzzb9rdd1m5n9l
Name:           frontweb
Service Mode:   Global
Placement:
UpdateConfig:
 Parallelism:   1
 On failure:    pause
 Monitoring Period: 5s
 Max failure ratio: 0
 Update order:      stop-first
RollbackConfig:
 Parallelism:   1
 On failure:    pause
 Monitoring Period: 5s
 Max failure ratio: 0
 Rollback order:    stop-first
ContainerSpec:
 Image:         nginx:latest@sha256:3e2ffcf0edca2a4e9b24ca442d227baea7b7f0e33ad654ef1eb806fbd9bedcf0
Resources:
Endpoint Mode:  vip
```

# 服务扩容缩容
集群必然涉及服务高可用，从而会有服务的扩容缩容

`将web服务扩容为3个task`

```shell
[root@docker-0 swarm]# docker service scale web=3
web scaled to 3
overall progress: 3 out of 3 tasks
1/3: running   [==================================================>]
2/3: running   [==================================================>]
3/3: running   [==================================================>]
verify: Service converged
```

`查看服务，可看到web服务的replicas副本数量为3个`

```shell
[root@docker-0 swarm]# docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE               PORTS
p2geazk5hdck        frontweb            global              3/3                 nginx:latest
plwzhwijh15j        web                 replicated          3/3                 nginx:latest
```

`可以看到三个节点上都运行了一个web服务(task)`

```shell
[root@docker-0 swarm]# docker service ps web
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
qjrl2qg04a3b        web.1               nginx:latest        docker-0            Running             Running 41 minutes ago
s6ejxq4xb1qo        web.2               nginx:latest        docker-2            Running             Running 3 minutes ago
5xim2g6fmom8        web.3               nginx:latest        docker-1            Running             Running 3 minutes ago
```
在默认情况下，管理节点也可以运行task，所以上面可以看到Manager节点上也运行了web服务

`将web服务缩容为2个`

```shell
[root@docker-0 swarm]# docker service scale web=2
web scaled to 2
overall progress: 2 out of 2 tasks
1/2: running   [==================================================>]
2/2: running   [==================================================>]
verify: Service converged
```

`查看运行的web容器`

```shell
[root@docker-0 swarm]# docker service ps web
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE               ERROR               PORTS
qjrl2qg04a3b        web.1               nginx:latest        docker-0            Running             Running about an hour ago
s6ejxq4xb1qo        web.2               nginx:latest        docker-2            Running             Running 9 minutes ago
```
当要让swarm的manager节点不运行容器的时候，只要更改节点的状态，从Active变成Drain即可；
如果在manager上运行容器，那么当manager宕机的时候，如果不是多节点的manager，会导致服务无法进行调度；

`将manager节点的状态修改为Drain状态，使得此节点不会运行相关的task任务`

```shell
[root@docker-0 swarm]# docker node update --availability drain docker-0
docker-0

# 查看节点的状态从active变成drain
[root@docker-0 swarm]# docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
dlr9h76ov43r7239sq3iksjyw *   docker-0            Ready               Drain               Leader              18.03.1-ce
wscczkpfb15c5knm6cr6lj712     docker-1            Ready               Active                                  18.03.1-ce
pgpnatt19yjti3eqy0gc3ebwg     docker-2            Ready               Active                                  18.03.1-ce
```

`本来运行在docker-0节点上的所有task会被关闭，然后mode=replicated的服务会自动迁移到其他的worker节点上(这一点有利也有弊)，mode=global的服务不会自动迁移`

```shell
[root@docker-0 swarm]# docker service ps web
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
q57mgwxfdxl0        web.1               nginx:latest        docker-1            Running             Running 2 minutes ago
qjrl2qg04a3b         \_ web.1           nginx:latest        docker-0            Shutdown            Shutdown 2 minutes ago
s6ejxq4xb1qo        web.2               nginx:latest        docker-2            Running             Running 18 minutes ago
```

# 自动故障转移
当运行在某节点上的task被关闭、或者节点宕机、或者网络抖动，mode=replicated的服务会自动迁移到其他的worker节点上(这一点有利也有弊)，mode=global的服务不会自动迁移；  
mode=replicated时的自动迁移，在生产环境中需要考虑的一个问题就是，如果出现了自动failover，那么其他的机器是否有足够的资源来创建这些容器，所以需要考虑node节点剩余资源的问题，例如cpu和内存；

场景模拟：
1、mode=replicated的web服务，在docker-1和docker-2这2个节点上分别运行着一个task；
2、模拟将docker-2节点进行宕机(关闭docker服务)，观察docker-1节点上将会有2个task运行；
3、恢复docker-2节点服务，观察docker-1节点上还是会有2个task运行，docker-2节点不会有task运行，可以看到并不会把之前迁移到docker-1上的服务自动迁移回来，所以这里也比较坑；

模拟宕机前：

| 服务名 | 节点主机名 | 运行状态 |
|--------|--------|--------|
| web.1 | docker-1 |Running|
| web.2 | docker-2 |Running|


模拟宕机后(docker-2宕机)：

| 服务名 | 节点主机名 | 运行状态 |
|--------|--------|--------|
| web.1 | docker-1 |Running|
| web.2 | docker-1 |Running|
| web.2 | docker-2 |Shutdown|

模拟恢复后：

| 服务名 | 节点主机名 | 运行状态 |
|--------|--------|--------|
| web.1 | docker-1 |Running|
| web.2 | docker-1 |Running|
| web.2 | docker-2 |Shutdown|


`查看服务分布`

```shell
[root@docker-0 swarm]# docker service ps web
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE             ERROR               PORTS
q57mgwxfdxl0        web.1               nginx:latest        docker-1            Running             Running 25 minutes ago
s6ejxq4xb1qo        web.2               nginx:latest        docker-2            Running             Running 42 minutes ago
```

`模拟docker-2宕机`

```shell
# 登陆docker-2节点，关闭docker服务
[root@docker-2 /]# systemctl stop docker
```

`查看节点信息，可以看到docker-2已经为down，也就是主机宕机`

```shell
[root@docker-0 swarm]# docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
dlr9h76ov43r7239sq3iksjyw *   docker-0            Ready               Drain               Leader              18.03.1-ce
wscczkpfb15c5knm6cr6lj712     docker-1            Ready               Active                                  18.03.1-ce
pgpnatt19yjti3eqy0gc3ebwg     docker-2            Down                Active                                  18.03.1-ce
```

`观察docker-1节点上将会有2个task运行，宕机的docker-2标记为shutdown`

```shell
[root@docker-0 swarm]# docker service ps web
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE             ERROR               PORTS
q57mgwxfdxl0        web.1               nginx:latest        docker-1            Running             Running 42 minutes ago
uqpzuj4tkwpd        web.2               nginx:latest        docker-1            Running             Running 2 minutes ago
s6ejxq4xb1qo         \_ web.2           nginx:latest        docker-2            Shutdown            Running 2 minutes ago
```

`将docker-2节点进行上线`

```shell
[root@docker-2 /]# systemctl start docker
```

`查看服务节点信息，docker-2状态变为ready,表示可以运行task任务`

```shell
[root@docker-0 swarm]# docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
dlr9h76ov43r7239sq3iksjyw *   docker-0            Ready               Drain               Leader              18.03.1-ce
wscczkpfb15c5knm6cr6lj712     docker-1            Ready               Active                                  18.03.1-ce
pgpnatt19yjti3eqy0gc3ebwg     docker-2            Ready               Active                                  18.03.1-ce
```

`再次查看web服务的分布，服务没有再迁移回docker-2节点`

```shell
[root@docker-0 swarm]# docker service ps web
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE                ERROR               PORTS
q57mgwxfdxl0        web.1               nginx:latest        docker-1            Running             Running about an hour ago
uqpzuj4tkwpd        web.2               nginx:latest        docker-1            Running             Running 16 minutes ago
s6ejxq4xb1qo         \_ web.2           nginx:latest        docker-2            Shutdown            Shutdown 2 minutes ago
```

# 访问服务
访问服务分为两种，对内、对外；  
对内：内部访问的服务，不对外开放端口  
对外：外部节点可访问，对外开放端口，也就是主机映射端口  

`创建2个副本的httpd服务`

```shell
[root@docker-0 swarm]# docker service create --name web --replicas=2 httpd
6mxfzdhrqeqgexmbyq5u7j6ds
overall progress: 2 out of 2 tasks
1/2: running   [==================================================>]
2/2: running   [==================================================>]
verify: Service converged
```
或者

```shell
[root@docker-0 swarm]# docker service create --name web --publish 8000:80 --replicas=2 httpd
```


`查看运行容器的主机`

```shell
[root@docker-0 swarm]# docker service ps web
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE                ERROR               PORTS
5fwbadj1s8gx        web.1               httpd:latest        docker-2            Running             Running about a minute ago
klab0nnwke22        web.2               httpd:latest        docker-1            Running             Running about a minute ago
```

`登陆节点服务，查看运行的容器`

```shell
[root@docker-1 /]# docker ps
CONTAINER ID        IMAGE               COMMAND              CREATED             STATUS              PORTS               NAMES
74429db851d3        httpd:latest        "httpd-foreground"   33 seconds ago      Up 13 seconds       80/tcp              web.2.klab0nnwke223tzztixiii1uq

[root@docker-2 /]# docker ps
CONTAINER ID        IMAGE               COMMAND              CREATED              STATUS              PORTS               NAMES
577c242824f0        httpd:latest        "httpd-foreground"   About a minute ago   Up About a minute   80/tcp              web.1.5fwbadj1s8gxx4smkils5lmu9
```

`查看运行容器的IP地址`

```shell
[root@docker-1 /]# docker exec 74429db851d3 ip addr show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
18: eth0@if19: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:ac:12:00:02 brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.2/16 brd 172.18.255.255 scope global eth0
       valid_lft forever preferred_lft forever

# 根据IP地址访问，只能在本节点上进行访问，属于内部网络，也就是docker_gwbrige网络
[root@docker-1 /]# curl 172.18.0.2
<html><body><h1>It works!</h1></body></html>

```

`添加主机映射端口，将服务对外开放，使外部能访问此服务`

```shell
[root@docker-0 swarm]# docker service update --publish-add 8000:80 web
web
overall progress: 2 out of 2 tasks
1/2: running   [==================================================>]
2/2: running   [==================================================>]
verify: Service converged

# 可以观察到，原来的容器被关闭，重新创建了新的容器
[root@docker-0 swarm]# docker service ps web
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
sad9u6d1gf1v        web.1               httpd:latest        docker-2            Running             Running 2 minutes ago
5fwbadj1s8gx         \_ web.1           httpd:latest        docker-2            Shutdown            Shutdown 2 minutes ago
nbxubcnx7dpy        web.2               httpd:latest        docker-1            Running             Running 2 minutes ago
klab0nnwke22         \_ web.2           httpd:latest        docker-1            Shutdown            Shutdown 2 minutes ago

# 无论是manager节点还是worker节点，均监听了8000端口，均可以访问
[root@docker-0 swarm]# curl 172.17.0.3:8000
<html><body><h1>It works!</h1></body></html>
[root@docker-0 swarm]# curl 172.17.0.4:8000
<html><body><h1>It works!</h1></body></html>
[root@docker-0 swarm]# curl 172.17.0.3:8000

# 集群中每个机器都会监听8000端口，无论是否运行了这个容器
[root@docker-0 swarm]# netstat -ntlp|grep 8000
tcp6       0      0 :::8000                 :::*                    LISTEN      66487/dockerd
```

以上使用的就是routing mesh功能，在swarm内部实现了负载均衡，使用的网络为swarm自动创建的overlay网络。  
当使用publish端口时，最大坏处是对外暴露了端口号，而且在使用的时候，如果进行了故障转移，那么多个服务运行在同一个主机上面，会造成端口占用冲突；

# 服务发现
docker-swarm原生提供了服务发现功能，也可以使用外部中间件，如consul/etcd/zookeeper;  

`创建overlay网络`  
要使用服务发现，需要相互通信的service必须属于同一个overlay网络,所以我们先创建一个新的overlay网络；

```shell
[root@docker-0 swarm]# docker network create --driver overlay web
s9jlkeot6iua2pzldff16dz8k
[root@docker-0 swarm]# docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
301fd3e9c939        bridge              bridge              local
5ab7c9aec44f        docker_gwbridge     bridge              local
9faeade5067d        host                host                local
k0m6xm3c1pgr        ingress             overlay             swarm
74eb81b4e335        none                null                local
s9jlkeot6iua        web                 overlay             swarm
```
默认的ingress没有提供服务发现，必须创建自己的overlay网络；

`部署service到新建的overlay`  
部署一个web服务，并将其挂载到新创建的overlay网络

```shell
[root@docker-0 swarm]# docker service create --name my-web --replicas=2 --network web httpd
qvws1y4kvkkiii9ny1tzmfrio
overall progress: 2 out of 2 tasks
1/2: running
2/2: running
verify: Service converged
```
部署一个test服务用于测试，挂载到同一个overlay网络  

```shell
[root@docker-0 swarm]# docker service create --name test --network web busybox sleep 10000000
0o216j6g3cny2ulrs9ncacr4r
overall progress: 1 out of 1 tasks
1/1: running
verify: Service converged

# sleep 10000000的作用是保持busybox容器处于运行状态，我们才能进入到容器访问my-web
[root@docker-0 swarm]# docker service ps test
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
z6ozex5qo747        test.1              busybox:latest      docker-1            Running             Running 29 seconds ago
```

`验证：`
通过docker service ps test确认test所在节点在docker-1;  
登陆到docker-1节点，并进入test容器，然后ping my-web

```shell
[root@docker-1 /]# docker ps
CONTAINER ID        IMAGE               COMMAND              CREATED             STATUS              PORTS               NAMES
4f23e905ab2b        busybox:latest      "sleep 10000000"     10 seconds ago      Up 7 seconds                            test.1.z6ozex5qo74781icswy4245pd
0af60046e58c        httpd:latest        "httpd-foreground"   3 minutes ago       Up 2 minutes        80/tcp              my-web.1.llob2wie0nx1x98m1vw780hcv

[root@docker-1 /]# docker exec test.1.z6ozex5qo74781icswy4245pd ping -c 2 my-web
PING my-web (10.0.0.5): 56 data bytes
64 bytes from 10.0.0.5: seq=0 ttl=64 time=0.112 ms
64 bytes from 10.0.0.5: seq=1 ttl=64 time=0.167 ms

--- my-web ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.112/0.139/0.167 ms
```
10.0.0.5是虚拟IP

`查看各个主机的IP地址`

```shell
[root@docker-1 /]# docker exec test.1.z6ozex5qo74781icswy4245pd nslookup tasks.my-web
Server:    127.0.0.11
Address 1: 127.0.0.11

Name:      tasks.my-web
Address 1: 10.0.0.6 my-web.2.zrj1b3nqaqfgmm3ipaj4grk33.web
Address 2: 10.0.0.7 my-web.1.llob2wie0nx1x98m1vw780hcv.web
```

`查看VIP地址`

```shell
[root@docker-1 /]# docker exec test.1.z6ozex5qo74781icswy4245pd nslookup my-web
Server:    127.0.0.11
Address 1: 127.0.0.11

Name:      my-web
Address 1: 10.0.0.5
```
对于服务使用者，test服务根本就不需要知道my-web副本的IP、VIP，只需直接用service的名字就能访问服务；

# 滚动更新service
当需要进行更新的时候，swarm可以设定相应的策略来进行更新，例如并行更新多少个容器，更新时间间隔等；  
滚动更新降低了应用更新的风险，如果某个副本更新失败，整个更新将暂停，其他副本则可以继续提供服务，同时，在更新的过程中，总是有副本在运行的，因此也保证了业务连续性；

swarm安装如下步骤进行滚动更新：  
* 停止第一个副本
* 调度任务。选择worker node
* 在worker上用新的镜像启动副本
* 如果更新成功则继续更新下一个，如果失败，暂停整个更新的过程

测试：创建2个副本的service，镜像使用httpd:2.2.31,然后将其更新到httpd:2.2.32

```shell
[root@docker-0 swarm]# docker service create --name web-update --replicas=2 httpd:2.2.31
yhjzc1lmkygvzk7zma6v7rwfy
overall progress: 2 out of 2 tasks
1/2: running   [==================================================>]
2/2: running   [==================================================>]
verify: Service converged

[root@docker-0 swarm]# docker service ps web-update
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE                ERROR               PORTS
pnss5ryum1f9        web-update.1        httpd:2.2.31        docker-1            Running             Running about a minute ago
o7fkw7jl78j2        web-update.2        httpd:2.2.31        docker-2            Running             Running about a minute ago

# 将web-update更新到httpd:2.2.32 (默认一次只更新一个副本，且副本间没有等待时间)
[root@docker-0 swarm]# docker service update --image httpd:2.2.32 web-update
web-update
overall progress: 2 out of 2 tasks
1/2: running   [==================================================>]
2/2: running   [==================================================>]
verify: Service converged

[root@docker-0 swarm]# docker service ps web-update
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE                 ERROR               PORTS
ri7pp1gwfpof        web-update.1        httpd:2.2.32        docker-1            Running             Running 8 seconds ago
pnss5ryum1f9         \_ web-update.1    httpd:2.2.31        docker-1            Shutdown            Shutdown about a minute ago
974r7xpufhry        web-update.2        httpd:2.2.32        docker-2            Running             Running about a minute ago
o7fkw7jl78j2         \_ web-update.2    httpd:2.2.31        docker-2            Shutdown            Shutdown 2 minutes ago
```

`更新前,先设置更新的并发数和更新的时间间隔，--update-parallelism设置并行更新的副本数，通过--update-delay指定滚动更新的时间间隔`

```shell
# 副本增加到6个，并且设置更新并发数为2个，时间间隔为1分5秒
[root@docker-0 swarm]# docker service update --replicas 6 --update-parallelism 2 --update-delay 1m05s web-update
web-update
overall progress: 6 out of 6 tasks
1/6: running   [==================================================>]
2/6: running   [==================================================>]
3/6: running   [==================================================>]
4/6: running   [==================================================>]
5/6: running   [==================================================>]
6/6: running   [==================================================>]
verify: Service converged

[root@docker-0 swarm]# docker service ps web-update
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE                ERROR               PORTS
ri7pp1gwfpof        web-update.1        httpd:2.2.32        docker-1            Running             Running 10 minutes ago
pnss5ryum1f9         \_ web-update.1    httpd:2.2.31        docker-1            Shutdown            Shutdown 11 minutes ago
974r7xpufhry        web-update.2        httpd:2.2.32        docker-2            Running             Running 11 minutes ago
o7fkw7jl78j2         \_ web-update.2    httpd:2.2.31        docker-2            Shutdown            Shutdown 12 minutes ago
l5gt07q396dw        web-update.3        httpd:2.2.32        docker-2            Running             Running about a minute ago
tj4fookdmlko        web-update.4        httpd:2.2.32        docker-1            Running             Running 49 seconds ago
ncyji5qju1co        web-update.5        httpd:2.2.32        docker-2            Running             Running 49 seconds ago
1x58y4zqqyq6        web-update.6        httpd:2.2.32        docker-1            Running             Running about a minute ago

#查看具体的信息，可以看到设置更新的参数
[root@docker-0 swarm]# docker service inspect web-update --pretty

ID:             yhjzc1lmkygvzk7zma6v7rwfy
Name:           web-update
Service Mode:   Replicated
 Replicas:      6
Placement:
UpdateConfig:
 Parallelism:   2
 Delay:         1m5s
 On failure:    pause
 Monitoring Period: 5s
 Max failure ratio: 0
 Update order:      stop-first
RollbackConfig:
 Parallelism:   1
 On failure:    pause
 Monitoring Period: 5s
 Max failure ratio: 0
 Rollback order:    stop-first
ContainerSpec:
 Image:         httpd:2.2.32@sha256:a28a1a54ee7c4e03249b5eb3fed0c387399ffa5bb422ab50cbb19ffde76e58e7
Resources:
Endpoint Mode:  vip
```

`将目前的6个副本从httpd:2.2.32, 更新到httpd:2.4.16`

```shell
#可以观察到，每次2个副本同时更新，1分05秒后进行下一组的2个副本更新
[root@docker-0 swarm]# docker service update --image httpd:2.4.16 web-update
web-update
overall progress: 6 out of 6 tasks
1/6: running
2/6: running
3/6: running
4/6: running
5/6: running
6/6: running
verify: Service converged
```

如果在中途更新失败，则暂停整个集群的更新；  
那么将会导致整个集群的副本版本不一致，这时需要rollback回滚；

## 更新回滚
swarm有个方便的回滚功能，如果更新后的效果不理想，可以通过--rollback快速恢复到更新之前的状态；
但是它只能回滚到上一次执行docker service update之前的状态，并不能无限制回滚；

```shell
[root@docker-0 swarm]# docker service update --rollback web-update
```
测试下来，这个回滚并不是很稳定，会有回滚失败导致容器副本数减少、有些副本回滚没成功问题；

# 节点标签
给node节点加label标签，用于控制task运行在哪些节点；  
生产环境上有时需要手动将容器分布在不同的节点；

`更新节点属性，为节点添加标签`

```shell
[root@docker-0 swarm]# docker node update --label-add ncname=docker-1-label docker-1
docker-1
[root@docker-0 swarm]# docker node update --label-add ncname=docker-2-label docker-2
docker-2
```

`查看设置的标签`

```shell
[root@docker-0 swarm]# docker node inspect docker-1 --pretty
ID:                     wscczkpfb15c5knm6cr6lj712
Labels:
 - ncname=docker-1-label
```

`根据label , 指定节点来运行相关的任务`

```shell
[root@docker-0 swarm]# docker service create --name web --constraint node.labels.ncname==docker-1-label --replicas 2 nginx
5xqg712i4ugcli6myq0u4pa75
overall progress: 2 out of 2 tasks
1/2: running   [==================================================>]
2/2: running   [==================================================>]
verify: Service converged

# 可以看到，运行在指定的机器上
[root@docker-0 swarm]# docker service ps web
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
g0n5bz0zrml1        web.1               nginx:latest        docker-1            Running             Running 13 seconds ago
jdaah28caxa9        web.2               nginx:latest        docker-1            Running             Running 13 seconds ago
```

`去掉指定的标签`

```shell
[root@docker-0 swarm]# docker service update --constraint-rm node.labels.ncname==docker-1-label web
web
overall progress: 2 out of 2 tasks
1/2: running
2/2: running
verify: Service converged
```

`服务迁移到另一台主机上`

```shell
[root@docker-0 swarm]# docker service update --constraint-add node.labels.ncname==docker-2-label web
web
overall progress: 2 out of 2 tasks
1/2: running
2/2: running
verify: Service converged

[root@docker-0 swarm]# docker service ps web
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE                 ERROR               PORTS
vhc4kvdlbvhd        web.1               nginx:latest        docker-2            Running             Running about a minute ago
g0n5bz0zrml1         \_ web.1           nginx:latest        docker-1            Shutdown            Shutdown about a minute ago
3uc0w4ozudfm        web.2               nginx:latest        docker-2            Running             Running 53 seconds ago
jdaah28caxa9         \_ web.2           nginx:latest        docker-1            Shutdown            Shutdown 55 seconds ago
```
这种修改也可以使用rollback进行回滚；
但我使用回滚命令，然而并没有什么卵用，留待以后再做验证
```shell
[root@docker-0 swarm]# docker service update --rollback web
```

# 健康检查
在企业生产环境中，合理的健康检查设置可以保证应用的可用性。现在很多应用框架已经内置了监控检查能力，比如Spring Boot Actuator。配合Docker内置的健康检测机制，可以非常简洁实现应用可用性监控，自动故障处理，和零宕机更新。

有两种方式：
* dockerfile 中加指令方式
* docker service create 参数指令方式

## dockerfile 中加指令方式
HEALTHCHECK 指令格式：
* HEALTHCHECK [选项] CMD <命令>：设置检查容器健康状况的命令
* HEALTHCHECK NONE：如果基础镜像有健康检查指令，使用这行可以屏蔽掉

注：在Dockerfile中 HEALTHCHECK 只可以出现一次，如果写了多个，只有最后一个生效。

使用包含 HEALTHCHECK 指令的dockerfile构建出来的镜像，在实例化Docker容器的时候，就具备了健康状态检查的功能。启动容器后会自动进行健康检查。

HEALTHCHECK 支持下列选项：
* --interval=<间隔>：两次健康检查的间隔，默认为 30 秒；
* --timeout=<间隔>：健康检查命令运行超时时间，如果超过这个时间，本次健康检查就被视为失败，默认 30 秒；
* --retries=<次数>：当连续失败指定次数后，则将容器状态视为 unhealthy，默认 3 次。
* --start-period=<间隔>: 应用的启动的初始化时间，在启动过程中的健康检查失效不会计入，默认 0 秒； (从17.05)引入

在 HEALTHCHECK [选项] CMD 后面的命令，格式和 ENTRYPOINT 一样，分为 shell 格式，和 exec 格式。命令的返回值决定了该次健康检查的成功与否：
* 0：成功；
* 1：失败；
* 2：保留值，不要使用

容器启动之后，初始状态会为 starting (启动中)。Docker Engine会等待 interval 时间，开始执行健康检查命令，并周期性执行。如果单次检查返回值非0或者运行需要比指定 timeout 时间还长，则本次检查被认为失败。如果健康检查连续失败超过了 retries 重试次数，状态就会变为 unhealthy (不健康)。

注：
* 一旦有一次健康检查成功，Docker会将容器置回 healthy (健康)状态
* 当容器的健康状态发生变化时，Docker Engine会发出一个 health_status 事件。

假设我们有个镜像是个最简单的 Web 服务，我们希望增加健康检查来判断其 Web 服务是否在正常工作，我们可以用 curl来帮助判断，其 Dockerfile 的 HEALTHCHECK 可以这么写：
```shell
FROM elasticsearch:5.5
 
HEALTHCHECK --interval=5s --timeout=2s --retries=12 \
  CMD curl --silent --fail localhost:9200/_cluster/health || exit 1
```

```shell
[root@docker-1 es]# ll
total 4
-rw-r--r-- 1 root root 148 Jun 28 07:48 Dockerfile

# 构建镜像
[root@docker-1 es]# docker build -t test/elasticsearch:5.5 .

# 运行镜像
[root@docker-1 es]# docker run --rm -d --name=elasticsearch test/elasticsearch:5.5

```

`查看Elasticsearch容器从 starting 状态进入了 healthy 状态`

```shell
[root@docker-1 es]# docker ps
CONTAINER ID        IMAGE                    COMMAND                  CREATED             STATUS                             PORTS                NAMES
f129ea7ebc6e        test/elasticsearch:5.5   "/docker-entrypoint.…"   47 seconds ago      Up 10 seconds (health: starting)   9200/tcp, 9300/tcp   elasticsearch
[root@docker-1 es]# docker ps
CONTAINER ID        IMAGE                    COMMAND                  CREATED              STATUS                    PORTS                NAMES
f129ea7ebc6e        test/elasticsearch:5.5   "/docker-entrypoint.…"   About a minute ago   Up 36 seconds (healthy)   9200/tcp, 9300/tcp   elasticsearch
```

`另外一种方法是在 docker run 命令中，直接指明healthcheck相关策略。`

```shell
$ docker run --rm -d \
    --name=elasticsearch \
    --health-cmd="curl --silent --fail localhost:9200/_cluster/health || exit 1" \
    --health-interval=5s \
    --health-retries=12 \
    --health-timeout=2s \
    elasticsearch:5.5

```

`为了帮助排障，健康检查命令的输出（包括 stdout 以及 stderr）都会被存储于健康状态里，可以用 docker inspect 来查看。我们可以通过如下命令，来获取过去5个容器的健康检查结果`

```shell
docker inspect --format='{{json .State.Health}}' elasticsearch

# 或者
docker inspect elasticsearch | jq ".[].State.Health"

# 或者
docker inspect elasticsearch
```

返回
```shell
{
  "Status": "healthy",
  "FailingStreak": 0,
  "Log": [
    {
      "Start": "2018-06-28T09:04:45.612342812Z",
	  "End": "2018-06-28T09:04:45.736592584Z",
      "ExitCode": 0,
      "Output": "..."
    },
    ...
}

```
建议在Dockerfile中声明相应的健康检查策略，这样可以方便镜像的使用

Docker社区为提供了一些包含健康检查的实例镜像，我们可以在如下项目中获取 https://github.com/docker-library/healthcheck

## docker service create 参数指令方式
在Docker 1.13之后，在Docker Swarm mode中提供了对健康检查策略的支持

可以在 docker service create 命令中指明健康检查策略

```shell
$ docker service create -d \
    --name=elasticsearch \
    --health-cmd="curl --silent --fail localhost:9200/_cluster/health || exit 1" \
    --health-interval=5s \
    --health-retries=12 \
    --health-timeout=2s \
    elasticsearch

```
在Swarm模式下，Swarm manager会监控服务task的健康状态，如果容器进入 unhealthy 状态，它会停止容器并且重新启动一个新容器来取代它。这个过程中会自动更新服务的 load balancer (routing mesh) 后端或者 DNS记录，可以保障服务的可用性。

在1.13版本之后，在服务更新阶段也增加了对健康检查的支持，这样在新容器完全启动成功并进入健康状态之前，load balancer/DNS解析不会将请求发送给它。这样可以保证应用在更新过程中请求不会中断。

# 常见问题
## 空间不足
如果哪天发现镜像无法构建，那一般都是空间问题；

方法一：暴力清除
```shell
# 查看哪个目录占用空间最大
du -sh /*
# 停止docker服务
systemctl stop docker
# 删除/var/lib/docker目录下的所有文件
rm -rf /var/lib/docker
# 启动docker服务
systemctl start docker
```
注意：docker 服务重启后，需要重新加入集群节点
```
docker swarm join --token SWMTKN-1-3d7ynvjvs9kqdqaut2sir2ky42uoiv54fmwef5s7odei33a031-3rcwr1styfzw8f8j3k9mvbz9c 172.17.0.2:2377
```

方法二： 转移/var/lib/docker目录  
略...
