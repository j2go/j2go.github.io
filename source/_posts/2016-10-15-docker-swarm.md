---
layout: post
title: "Docker Swarm 试用"
date: 2016-10-15
tags: Swarm
categories: Docker
---
## 环境
> 系统 Ubuntu 14.04, Docker version: 1.12

## 创建 swarm
节点操作   
机器 a
```
root@ide-a:~# docker swarm init
Error response from daemon: could not choose an IP address to advertise since this system has multiple addresses on different interfaces (10.211.55.16 on eth0 and 192.168.3.10 on eth1) - specify one with --advertise-addr

root@ide-a:~# docker swarm init --advertise-addr 192.168.3.10
Swarm initialized: current node (e6q1r4qw1jatxvqk056fu6502) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-6cvv0fm296x7c686a8ojxie33ob502hjdxickcj7ycitwqd2ej-bch1g33glv0ktox9qg0q7qshj \
    192.168.3.10:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
root@ide-a:~# docker node ls
ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
e6q1r4qw1jatxvqk056fu6502 *  ide-a     Ready   Active        Leader
```
机器 b
```
vagrant@ide-b:~$ docker swarm join \
>     --token SWMTKN-1-6cvv0fm296x7c686a8ojxie33ob502hjdxickcj7ycitwqd2ej-bch1g33glv0ktox9qg0q7qshj \
>     192.168.3.10:2377
This node joined a swarm as a worker.
```
同样添加一个机器 c

然后在机器 a 上看
```
vagrant@ide-a:~$ docker node ls
ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
2mcmsgvu4nvgymaqaexnevxn3    ide-b     Ready   Active
6arg09xduwb65r6zaof61tjzk    ide-c     Ready   Active
e6q1r4qw1jatxvqk056fu6502 *  ide-a     Ready   Active        Leader
```
如果关闭一个机器
```
vagrant@ide-a:~$ docker node ls
ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
2mcmsgvu4nvgymaqaexnevxn3    ide-b     Down    Active
6arg09xduwb65r6zaof61tjzk    ide-c     Ready   Active
e6q1r4qw1jatxvqk056fu6502 *  ide-a     Ready   Active        Leader
```
把一个结点变成 manager
```
vagrant@ide-a:~$ docker node promote 2mcmsgvu4nvgymaqaexnevxn3
Node 2mcmsgvu4nvgymaqaexnevxn3 promoted to a manager in the swarm.

ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
2mcmsgvu4nvgymaqaexnevxn3    ide-b     Ready   Active        Reachable
6arg09xduwb65r6zaof61tjzk    ide-c     Ready   Active
e6q1r4qw1jatxvqk056fu6502 *  ide-a     Ready   Active        Leader
```
模拟 a 机器宕机

但是 b 机器始终没有变成 manager
```
vagrant@ide-b:~$ docker node ls
Error response from daemon: rpc error: code = 2 desc = raft: no elected cluster leader
```
当重启 a 机器节点时 b 就成 Leader  了
```
ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
2mcmsgvu4nvgymaqaexnevxn3 *  ide-b     Ready   Active        Leader
6arg09xduwb65r6zaof61tjzk    ide-c     Ready   Active
e6q1r4qw1jatxvqk056fu6502    ide-a     Ready   Active        Reachable
```
由此怀疑内部的 raft 算法是一定要收到回复才能确定自己是 Leader
 把 c 也设一下 manager 然后关闭 b 看看
```
vagrant@ide-a:~$ docker node promote ide-c
Node ide-c promoted to a manager in the swarm.
vagrant@ide-a:~$ docker node ls
ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
2mcmsgvu4nvgymaqaexnevxn3    ide-b     Ready   Active        Leader
6arg09xduwb65r6zaof61tjzk    ide-c     Ready   Active        Reachable
e6q1r4qw1jatxvqk056fu6502 *  ide-a     Ready   Active        Reachable

vagrant@ide-a:~$ docker node ls
ID                           HOSTNAME  STATUS   AVAILABILITY  MANAGER STATUS
2mcmsgvu4nvgymaqaexnevxn3    ide-b     Unknown  Active        Unreachable
6arg09xduwb65r6zaof61tjzk    ide-c     Ready    Active        Reachable
e6q1r4qw1jatxvqk056fu6502 *  ide-a     Ready    Active        Leader
```
ok，这下就是无缝切换了。

在swarm 上运行一个应用
```
vagrant@ide-a:~$ docker service create --replicas 1 --name helloworld alpine ping docker.com
exzsbuclde7fmxr74days2v7h
vagrant@ide-a:~$ docker node ps ide-a
ID  NAME  IMAGE  NODE  DESIRED STATE  CURRENT STATE  ERROR
vagrant@ide-a:~$ docker node ps ide-b
ID                         NAME          IMAGE   NODE   DESIRED STATE  CURRENT STATE             ERROR
cvsbavho7pqvabhe1pk4r5zik  helloworld.1  alpine  ide-b  Running        Preparing 13 seconds ago
vagrant@ide-a:~$ docker node ps ide-c
ID  NAME  IMAGE  NODE  DESIRED STATE  CURRENT STATE  ERROR

vagrant@ide-a:~$ docker service ls
ID            NAME        REPLICAS  IMAGE   COMMAND
exzsbuclde7f  helloworld  1/1       alpine  ping docker.com
vagrant@ide-a:~$ docker service inspect --pretty exzsbuclde7f
ID:		exzsbuclde7fmxr74days2v7h
Name:		helloworld
Mode:		Replicated
Replicas:	1
Placement:
UpdateConfig:
Parallelism:	1
On failure:	pause
ContainerSpec:
Image:		alpine
Args:		ping docker.com
Resources:
```
查看在哪台机器上
```
vagrant@ide-a:~$ docker service ps helloworld
ID                         NAME          IMAGE   NODE   DESIRED STATE  CURRENT STATE          ERROR
cvsbavho7pqvabhe1pk4r5zik  helloworld.1  alpine  ide-b  Running        Running 5 minutes ago
```
扩展5个实例
```
vagrant@ide-b:~$ docker service scale helloworld=5
helloworld scaled to 5
vagrant@ide-b:~$ docker service ps helloworld
ID                         NAME          IMAGE   NODE   DESIRED STATE  CURRENT STATE             ERROR
cvsbavho7pqvabhe1pk4r5zik  helloworld.1  alpine  ide-b  Running        Running 15 minutes ago
3z6yoie0qs5o388win82g9s3i  helloworld.2  alpine  ide-c  Running        Preparing 16 seconds ago
dbltvky8b1omkhfsgetudfzix  helloworld.3  alpine  ide-b  Running        Preparing 15 seconds ago
0zz270134yk761b61btj2u7vd  helloworld.4  alpine  ide-a  Running        Preparing 16 seconds ago
7cqitdly74zcrf6vy4pzhbw2u  helloworld.5  alpine  ide-c  Running        Preparing 16 seconds ago
```
删除 service
```
vagrant@ide-b:~$ docker service rm helloworld
helloworld
vagrant@ide-b:~$ docker service ps helloworld
Error: No such service: helloworld
```

## 检验 swarm 是否保证实例个数
开启3个实例
```
vagrant@ide-b:~$ docker service create --replicas 3 --name helloworld alpine ping docker.com
Dt81b618tz8yu8g5opw5990di

vagrant@ide-b:~$ docker service ps helloworld
ID                         NAME          IMAGE   NODE   DESIRED STATE  CURRENT STATE             ERROR
6drq6vomc15ils31ymr6lhgqk  helloworld.1  alpine  ide-a  Running        Preparing 30 seconds ago
0j3m7oqf2cxxv1dof7q2fbhzw  helloworld.2  alpine  ide-c  Running        Running 23 seconds ago
7khhv039gw3a9h4p3qaorbkg3  helloworld.3  alpine  ide-b  Running        Running 21 seconds ago
```
关闭 c 机器
```
vagrant@ide-b:~$ docker service ps helloworld
ID                         NAME              IMAGE   NODE   DESIRED STATE  CURRENT STATE               ERROR
6drq6vomc15ils31ymr6lhgqk  helloworld.1      alpine  ide-a  Running        Running 53 seconds ago
85d8d1tuus0qmikwtv0qi6ulo  helloworld.2      alpine  ide-b  Running        Preparing 7 seconds ago
0j3m7oqf2cxxv1dof7q2fbhzw   \_ helloworld.2  alpine  ide-c  Shutdown       Running about a minute ago
7khhv039gw3a9h4p3qaorbkg3  helloworld.3      alpine  ide-b  Running        Running about a minute ago
```
OK，确实有3个实例在，那么再开启 c 机器
```
vagrant@ide-b:~$ docker node ls
ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
2mcmsgvu4nvgymaqaexnevxn3 *  ide-b     Ready   Active        Reachable
6arg09xduwb65r6zaof61tjzk    ide-c     Ready   Active
e6q1r4qw1jatxvqk056fu6502    ide-a     Ready   Active        Leader
vagrant@ide-b:~$ docker service ps helloworld
ID                         NAME              IMAGE   NODE   DESIRED STATE  CURRENT STATE               ERROR
6drq6vomc15ils31ymr6lhgqk  helloworld.1      alpine  ide-a  Running        Running 2 minutes ago
85d8d1tuus0qmikwtv0qi6ulo  helloworld.2      alpine  ide-b  Running        Running about a minute ago
0j3m7oqf2cxxv1dof7q2fbhzw   \_ helloworld.2  alpine  ide-c  Shutdown       Complete 19 seconds ago
7khhv039gw3a9h4p3qaorbkg3  helloworld.3      alpine  ide-b  Running        Running 3 minutes ago
```
没有变化，在 c 上强行开起来 ps 结果也没发生变化
删除服务
```
vagrant@ide-b:~$  docker service rm helloworld
helloworld
vagrant@ide-b:~$ docker service ps helloworld
Error: No such service: helloworld
```
 c 机器上依然有这个 不按规则 的 container存在
```
vagrant@ide-c:/data/coding-ide-home$ docker ps
CONTAINER ID        IMAGE                                                     COMMAND                  CREATED             STATUS              PORTS                                                                NAMES
ffe29f05f2e7        alpine:latest                                             "ping docker.com"        9 minutes ago       Up About a minute                                                                        helloworld.2.0j3m7oqf2cxxv1dof7q2fbhzw
```
测试过如果没有人为启动起来在用 service rm  指令删除后该异常 container 也退出了
## 关闭一个节点
但是该节点的 docker  deman没有停掉
准确的说是停用一个节点，设置了一个状态
```
$ docker node update --availability drain ide-c
```
复活
```
$ docker node update --availability active ide-c
```
## 测试滚动更新
创建一个滚动更新的服务
```
vagrant@ide-a:~$ docker service create --replicas 3 --name redis --update-delay 10s redis:3.0.6
c1cieiareqz8l43ckr8ekstc9
vagrant@ide-a:~$ docker service ps redis
ID                         NAME     IMAGE        NODE   DESIRED STATE  CURRENT STATE             ERROR
dv0bqyp5j8rt2qhdb67vk1ux6  redis.1  redis:3.0.6  ide-a  Running        Preparing 11 seconds ago
ednp70fzj2nocae2gismqgx3c  redis.2  redis:3.0.6  ide-c  Running        Preparing 11 seconds ago
1m0fnh3qhkj1rfpufdkmn3cqd  redis.3  redis:3.0.6  ide-b  Running        Preparing 13 seconds ago
```
作用是神马？
为了多实例的平滑升级
```
vagrant@ide-a:~$ docker service ps redis
ID                         NAME         IMAGE        NODE   DESIRED STATE  CURRENT STATE                 ERROR
dv0bqyp5j8rt2qhdb67vk1ux6  redis.1      redis:3.0.6  ide-a  Running        Preparing 4 minutes ago
ednp70fzj2nocae2gismqgx3c  redis.2      redis:3.0.6  ide-c  Running        Running 3 minutes ago
0rgbjgr5cf8gz7cyt0hhhgin4  redis.3      redis:3.0.7  ide-a  Running        Preparing about a minute ago
1m0fnh3qhkj1rfpufdkmn3cqd   \_ redis.3  redis:3.0.6  ide-b  Shutdown       Shutdown about a minute ago
```
不过好像测试过程中有点问题，只升级了一个节点就停止了

[Swarm 集群管理][1]

创建对外服务的实例
```
vagrant@ide-a:~$ docker service create --name my_web --replicas 3 --publish 2345:80 nginx
6wud0bt1qbs6upptvddxvkc4k
vagrant@ide-a:~$ docker service ps my_web
ID                         NAME      IMAGE  NODE   DESIRED STATE  CURRENT STATE            ERROR
cf5co7yx4x79cn2ruirothgi1  my_web.1  nginx  ide-a  Running        Preparing 4 seconds ago
3ibjg29g7nq6j1a2533neerb5  my_web.2  nginx  ide-c  Running        Preparing 4 seconds ago
096muixt5lmd5c0oedrwkzg9y  my_web.3  nginx  ide-b  Running        Preparing 5 seconds ago
```
能访问的地址是最开始设置的 --advertise-addr 的地址
```
vagrant@ide-a:~$ curl http://192.168.3.10:2345
```
**一个问题**: 对外有一个 端口 提供服务，如果这个主 IP 的节点挂掉了呢？

测试显示绑定一个端口对外服务，访问集群中所有 ip +该端口 均能得到结果
## 能否在指定节点上创建 container？

有一种方式在每个可用节点上运行一个 service
```
docker service create --name myservice --mode global alpine top
```
实验结果表明 swarm 都是负载均衡的，即使使用 global 模式，访问其中一个指定 ip 的请求也会均衡转发到各个节点中去

  [1]: https://docs.docker.com/engine/swarm/admin_guide/