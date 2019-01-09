---
title: docker save-load and export-import
comments: true
date: 2018-07-6 16:12:00
tags:
  - docker  
categories: 容器
---

# docker 镜像导入导出
两组导入导出命令：  
* docker save/load
* docker export/import

两种使用场景(两者可以结合使用)： 

| 命令 | 场景 |
|--------|--------|
| save/load |如果部署的节点服务器不能连外部或者内部镜像仓库，则可以docker save镜像打包，然后拷贝上传到要部署的节点服务其上，使用docker load载入|
|export/import|用于制作基础镜像，比如容器启动后，在容器中安装一些软件或者环境的设置，使用docker export导出修改后的容器，然后分发给其他人使用(比如作为基础的开发环境)|

也可以使用docker commit命令，提交修改后的容器，并上传至镜像仓库;  
注意：用docker commit命令打包的镜像，比Dockerfile方式打出的镜像大(镜像分层问题)

# 测试环境准备
步骤：
* 构建一个centos镜像
* 进入容器安装软件
* 配置防火墙
* 提交容器修改

构建镜像、运行容器、进入容器后台

```shell
$ git clone https://github.com/smalldok/docker-based-tools.git
$ cd dind_centos
$ docker build -t dind_centos .
$ docker run --name docker-0 --privileged -d dind_centos
$ docker exec -it -u root docker-0 /bin/bash
```

安装软件

```shell
# 安装支持ifconfig命令的工具
yum install net-tools

# 安装firewalld
yum install firewalld firewall-config
systemctl start  firewalld # 启动

```

开放docker-swarm需要的端口

```shell
# 开放防火墙端口
firewall-cmd --permanent --add-port=2377/tcp
firewall-cmd --permanent --add-port=7946/tcp
firewall-cmd --permanent --add-port=7946/udp
firewall-cmd --permanent --add-port=4789/udp
firewall-cmd --reload # 永久打开端口需要reload一下
firewall-cmd --list-all # 查看防火墙，添加的端口也可以看到
```

# docker export/import
* docker export  
docker export是将container的文件系统打包; 
* docker import  
docker import将container导入后成为一个Image,而不是恢复为一个container;  
可指定image[:tag]，为镜像指定新名称；如果名称相同，则会覆盖老的镜像；


`docker export 导出容器`

```shell
$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                                        NAMES
1eb256b1d4b1        dind_centos         "wrapdocker /usr/sbi…"   About an hour ago   Up About an hour       docker-0

$ docker export 1eb256b1d4b1 > dind_centos.tar
```

`docker import 导入`

```shell
$ docker import dind_centos.tar dind_centos:1.0
sha256:4e8612dad5ba4230e618a424727a679e0f52f7a1fcfb16bbfe5626338784d404

$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
dind_centos         1.0                 4e8612dad5ba        20 seconds ago      539MB
dind_centos         latest              9031f35e0ec5        4 days ago          550MB
centos              latest              49f7960eb7e4        3 weeks ago         200MB
```

`查看之前对容器的修改是否还在`

```shell
# 运行容器
$ docker run --name docker-test --privileged -d dind_centos:1.0 wrapdocker /usr/sbin/init
960ecfc2f531533ba36c3cefb00fa9f784cdafd14a898577c52262f03d3faede

# 查看ifconfig是否生效
$ ifconfig
```

# docker save/load

* docker save  
docker save 可以对image或者container打包；对container打包，其实打的是容器背后的image;  
docker save 可以用来将一个或者多个Image打包，如： 
```
# 打包之后的test.tar包含nginx:1.0 httpd:1.4这两个镜像
docker save -o test.tar nginx:1.0 httpd:1.4
```

* docker load  
```
# 该命令会把nginx:1.0 httpd:1.4载入进来，如果本地已经存在这两个镜像，会被覆盖
docker load -i test.tar
```

