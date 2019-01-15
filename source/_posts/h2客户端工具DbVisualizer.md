---
title: h2客户端工具DbVisualizer
comments: true
tags:
  - H2
  - 嵌入式数据库
categories: 嵌入式数据库
abbrlink: 31153
date: 2017-08-22 23:13:00
---


## 引言
> 一般springboot中启动h2，即可通过网页访问控制台，来操作H2；  
如，`http://localhost:9088/xxxserver/h2-console`  
但是要启动项目才能访问，极大的不方便；  
因此需要第三方工具直接打开`mychannel.mv.db`文件，像在console上操作一样；

## 准备
* 工具版本：
`DbVisualizer 9.0.7`  
网上有破解的，它还可连接所有主流关系型数据库(我日常就用的它)  
* 项目中使用的h2版本：`h2-1.4.193`

## 使用
* 假设`mychannel.mv.db`文件在`/aaa/bbb`目录  
=>  /aaa/bbb/mychannel.mv.db
* 打开DbVisualizer向导,创建一个H2连接：   
Database Type: `H2`  
Driver(JDBC)： `H2 embedded`  
Database filename: `/aaa/bbb/mychannel`  
Database Userid: 项目里面设置的用户名   
Database Password: 项目里面设置的密码

## 注意事项
* DbVisualizer 的H2驱动问题  
DbVisualizer 自己会下载数据库驱动，但自带的h2驱动为 1.3 的版本，目前最新h2版本为 1.4 （因为1.3版本默认是 dbname.h2.db 形式的，用1.3驱动无法正确打开最新的 dbname.mv.db 形式的数据库文件，因为默认后缀不同，连接URL带上数据库名，dbvisualizer 会自动认为是 dbname.h2.db 由于文件不存在，所以变成是新建一个 dbname.h2.db 数据库文件了）  
* 解决办法  
把最新版本的h2的jar包(本地maven仓库中拿)，复制到 DbVisualizer 安装目录下的 DbVisualizer\jdbc\h2 中，把原有目录中的h2.jar 删掉，把最新版本的h2 jar 包命名为 h2.jar 替换原有的。  
这样就能识别新版本的h2，打开已经创建了的 *.mv.db 了