---
layout: post
title: "Centos6.5 搭建 nexus + maven"
date: 2016-04-29
tags: Maven
categories: Java
---

# 安装MAVEN
>\#wget http://mirror.cc.columbia.edu/pub/software/apache/maven/maven-3/3.3.9/binaries/apache-maven-3.3.9-bin.tar.gz
>\#tar -zxvf apache-maven-3.3.9-bin.tar.gz
>\#mkdir /usr/local/maven
>\#mv -rf apache-maven-3.3.9/* /usr/local/maven/

### 修改环境配置
>\# vi /etc/profile

*加入*

>export M2_HOME=/usr/local/maven
>export PATH=${M2_HOME}/bin:${PATH}
>\# source /etc/profile

### 检查结果

>\# mvn -v
>Apache Maven 3.3.9 (bb52d8502b132ec0a5a3f4c09453c07478323dc5; 2015-11-11T00:41:47+08:00)
>Maven home: /usr/local/maven
>Java version: 1.7.0_79, vendor: Oracle Corporation
>Java home: /usr/java/jdk1.7.0_79/jre
>Default locale: zh_CN, platform encoding: UTF-8
>OS name: "linux", version: "2.6.32-431.el6.x86_64", arch: "amd64", family: "unix"




# 安装nexus
*需要翻墙才能下*
[nexus-2.13.0-01-bundle.tar.gz](https://sonatype-download.global.ssl.fastly.net/nexus/oss/nexus-2.13.0-01-bundle.tar.gz)

*然后解压*
>\#tar -zxvf nexus-2.13.0-01-bundle.tar.gz

产生如下两个目录
>drwxr-xr-x 8 1001 1001       4096 4月  28 16:06 nexus-2.13.0-01
>drwxr-xr-x 3 1001 1001       4096 4月  12 22:21 sonatype-work

*直接去运行会报错*
>\# nexus-2.13.0-01/bin/nexus start
>Starting Nexus OSS...
>Failed to start Nexus OSS.

stackoverflow上找到了答案，不能用root用户运行，新建并切换用户之...

_还要切换目录owner_
>\#chown -R rd:rd nexus-2.13.0-01
>\#chown -R sonatype-work

### 接下来就可以运行了

>\#nexus start

打开 [http://localhost:8081/nexus](http://localhost:8081/nexus) 即可看到nexus的管理页面

_默认账户密码_admin admin123