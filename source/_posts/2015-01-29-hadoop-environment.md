---
layout: post
title: "Hadoop环境配置遇到的问题"
date: 2015-01-29
tags: hadoop
categories: hadoop
---
>千不该万不该，换个题目跳个大坑，hadoop 没两把刷子还真玩不动。

## Hadoop环境配置遇到的问题

### 环境    
>系统：Windows7   
>VM虚拟机：ubuntu14.04 i386    
>jdk : jdk1.7.0_71   
 
### 参考链接：    
>http://www.cnblogs.com/kinglau/p/3794433.html   
>http://www.cnblogs.com/kinglau/p/3796164.html
> 
>http://blog.sina.com.cn/s/blog_675e4f240102uwim.html   
>http://blog.sina.com.cn/s/blog_675e4f240102uwpe.html
 
## 遇到的问题及解决办法：
 
### 单机配置时

#### 设置默认root密码   
root密码找回:
```
$sudo passwd root 
```
输入你安装时用户的密码，设置root密码。
 
#### 安装jdk时配置了环境还要替换一下系统的环境才能使用 jps 命令

  将系统默认的jdk修改过来
```
sudo update-alternatives --install /usr/bin/java java /usr/lib/jvm/jdk1.8.0_05/bin/java 300
sudo update-alternatives --install /usr/bin/javac javac /usr/lib/jvm/jdk1.8.0_05/bin/javac 300
sudo update-alternatives --config java 
sudo update-alternatives --config javac
```

#### 新建的hadoop用户，需要解除 /usr/local/hadoop 目录权限限制
```
sudo chmod a+rwx /usr/local/hadoop
```
#### 以下操作需要在hadoop用户下使用（不知道为什么）
执行启动命令：
```
sbin/start-dfs.sh    
sbin/start-yarn.sh  
```
namenode没有启动
```
Cannot create directory /usr/hadoop/tmp/hdfs/name/current解决方案:
hadoop@ubuntu:/usr/local/hadoop$ sudo chown -R hadoop:hadoop hdfs
```
再执行格式化:
```
bin/hfds namenode -format
hadoop@ubuntu:/usr/local/hadoop$ ls hdfs/name
current
hadoop@ubuntu:/usr/local/hadoop$ sbin/start-all.sh 
```
### 配置集群时

在验证前，需要做两件事儿。第一件事是修改文件"authorized_keys"权限（权限的设置非常重要，因为不安全的设置，会让你不能使用 ssh 功能），另一件事儿是用root用户设置"/etc/ssh/sshd_config"的内容,使机器间可以无密码登陆访问。

#### 修改文件"authorized_keys"
```
chmod 600 ~/.ssh/authorized_keys
```