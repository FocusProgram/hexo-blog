---
title: Zookeeper
date: 2020-07-11 01:53:08
tags: 
 - SpringCloud 
 - 微服务
categories: 微服务
index_img: https://gitee.com/FocusProgram/PicGo/raw/master/20200308214002.png
---

**Zookeeper**

---

# 1.什么是Zookeeper?

> [Zookeeper](https://github.com/apache/zookeeper) 是一个用于维护配置信息、命名、提供分布式同步和提供组服务的集中服务。 所有这些类型的服务都以某种形式被分布式应用程序所使用。 每次实现它们时，都有大量的工作需要去修复那些不可避免的 bug 和竞争条件。 由于实现这些类型的服务很困难，应用程序最初通常会对它们进行缩减，这使得它们在出现变更时变得脆弱，难以管理。 即使正确地执行，在部署应用程序时，这些服务的不同实现也会导致管理复杂性。Zookeeper 旨在将这些不同服务的本质提炼为一个非常简单的接口，以集中化的协调服务。 服务本身是分布式的，并且高度可靠。 服务将实现共识、组管理和存在协议，这样应用程序就不需要自己实现它们。 这些应用程序的特定用途将包括动物园管理员的特定组件和应用程序特定约定的混合。 Zookeeper Recipes 展示了如何使用这个简单的服务来构建更强大的抽象。

# 2. 搭建Zookeeper

> 根据自己选择，下载相应的Zookeeper版本 [下载地址](http://zookeeper.apache.org/releases.html)

```linux

$ tar -zxvf apache-zookeeper-3.6.0-bin.tar.gz

$ cd /apache-zookeeper-3.6.0-bin

$ mkdir data #创建data目录

$ vim /apache-zookeeper-3.6.0-bin/conf/zoo.cfg
```

编辑配置文件如下：

```linux
#ZK中的一个时间单元。ZK中所有时间都是以这个时间单元为基础，进行整数倍配置的。例如，session的最小超时时间是2*tickTime
tickTime=2000

#Follower在启动过程中，会从Leader同步所有最新数据，然后确定自己能够对外服务的起始状态。Leader允许F在 initLimit 时间内完成这个工作。通常情况下，我们不用太在意这个参数的设置。如果ZK集群的数据量确实很大了，F在启动的时候，从Leader上同步数据的时间也会相应变长，因此在这种情况下，有必要适当调大这个参数了
initLimit=10

#在运行过程中，Leader负责与ZK集群中所有机器进行通信，例如通过一些心跳检测机制，来检测机器的存活状态。如果L发出心跳包在syncLimit之后，还没有从F那里收到响应，那么就认为这个F已经不在线了。注意：不要把这个参数设置得过大，否则可能会掩盖一些问题
syncLimit=5

#存储快照文件snapshot的目录。默认情况下，事务日志也会存储在这里。建议同时配置参数dataLogDir, 事务日志的写性能直接影响zk性能
dataDir=/data/apache-zookeeper-3.6.0-bin/data

#客户端连接server的端口，即对外服务端口，一般默认为2181
clientPort=2181 
```

启动服务端

```linux
$ cd /data/apache-zookeeper-3.6.0-bin/bin/

$ ./zkServer.sh start
```

显示如下，则说明启动成功：

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200308214002.png)

# 3. 集群搭建Zookeeper

## 3.1 集群服务器所需配置

| hostname   | ipaddress         | mask             | gateway         | port |
|------------|-------------------|------------------|-----------------|------|
| master     | 192\.168\.80\.130 | 255\.255\.255\.0 | 192\.168\.80\.2 | 2181 |
| slave\-one | 192\.168\.80\.131 | 255\.255\.255\.0 | 192\.168\.80\.2 | 2181 |
| slave\-two | 192\.168\.80\.132 | 255\.255\.255\.0 | 192\.168\.80\.2 | 2181 |

## 3.2 集群数量为什么至少是三台，并且最好为奇数？

> zookeeper 集群通常是用来对用户的分布式应用程序提供协调服务的，为了保证数据的一致性，对 zookeeper 集群进行了这样三种角色划分：leader、follower、observer分别对应着总统、议员和观察者。

* **总统（leader）：负责进行投票的发起和决议，更新系统状态。**

* **议员（follower）：用于接收客户端请求并向客户端返回结果以及在选举过程中参与投票。**

* **观察者（observer）：也可以接收客户端连接，将写请求转发给leader节点，但是不参与投票过程，只同步leader的状态。通常对查询操作做负载。**

> 我们知道，在每台机器数据保持一致的情况下，zookeeper集群可以保证，客户端发起的每次查询操作，集群节点都能返回同样的结果。
>
> 但是对于客户端发起的修改、删除等能改变数据的操作呢？集群中那么多台机器，你修改你的，我修改我的，最后返回集群中哪台机器的数据呢？
>
> 这就是一盘散沙，需要一个领导，于是在zookeeper集群中，leader的作用就体现出来了，只有leader节点才有权利发起修改数据的操作，而follower节点即使接收到了客户端发起的修改操作，也要将其转交给leader来处理，leader接收到修改数据的请求后，会向所有follower广播一条消息，让他们执行某项操作，follower 执行完后，便会向 leader 回复执行完毕。当 leader 收到半数以上的 follower 的确认消息，便会判定该操作执行完毕，然后向所有 follower 广播该操作已经生效。
>
> 所以zookeeper集群中leader是不可缺少的，但是 leader 节点是怎么产生的呢？其实就是由所有follower 节点选举产生的，讲究民主嘛，而且leader节点只能有一个，毕竟一个国家不能有多个总统。
>
> 这个时候回到我们的小标题，为什么 zookeeper 节点数是奇数，我们下面来一一来说明：

* **容错率**

>   首先从容错率来说明：（需要保证集群能够有半数进行投票）
>
>　　2台服务器，至少2台正常运行才行（2的半数为1，半数以上最少为2），正常运行1台服务器都不允许挂掉，但是相对于 单节点服务器，2台服务器还有两个单点故障，所以直接排除了。
>
>　　3台服务器，至少2台正常运行才行（3的半数为1.5，半数以上最少为2），正常运行可以允许1台服务器挂掉
>
>　　4台服务器，至少3台正常运行才行（4的半数为2，半数以上最少为3），正常运行可以允许1台服务器挂掉
>
>　　5台服务器，至少3台正常运行才行（5的半数为2.5，半数以上最少为3），正常运行可以允许2台服务器挂掉

* **防脑裂**

>　　脑裂集群的脑裂通常是发生在节点之间通信不可达的情况下，集群会分裂成不同的小集群，小集群各自选出自己的leader节点，导致原有的集群出现多个leader节点的情况，这就是脑裂。
>
>　　3台服务器，投票选举半数为1.5，一台服务裂开，和另外两台服务器无法通行，这时候2台服务器的集群（2票大于半数1.5票），所以可以选举出leader，而 1 台服务器的集群无法选举。
>
>　　4台服务器，投票选举半数为2，可以分成 1,3两个集群或者2,2两个集群，对于 1,3集群，3集群可以选举；对于2,2集群，则不能选择，造成没有leader节点。
>
>　　5台服务器，投票选举半数为2.5，可以分成1,4两个集群，或者2,3两集群，这两个集群分别都只能选举一个集群，满足zookeeper集群搭建数目。
>
>　　以上分析，我们从容错率以及防止脑裂两方面说明了3台服务器是搭建集群的最少数目，4台发生脑裂时会造成没有leader节点的错误。


## 3.3 集群配置文件

### 3.3.1 创建myid文件

```linux
$ echo 0 > /data/apache-zookeeper-3.6.0-bin/data/myid #192.168.80.130

$ echo 1 > /data/apache-zookeeper-3.6.0-bin/data/myid #192.168.80.131

$ echo 2 > /data/apache-zookeeper-3.6.0-bin/data/myid #192.168.80.132
```

### 3.3.2 编辑配置文件zoo.cfg

```linux
tickTime=2000

initLimit=10

syncLimit=5

dataDir=/data/apache-zookeeper-3.6.0-bin/data

clientPort=2181

server.0=192.168.80.130:2888:3888
server.1=192.168.80.131:2888:3888
server.2=192.168.80.132:2888:3888
```

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200309102548.png)

> server.A=B:C:D
>
>　　　　A：其中 A 是一个数字，表示这个是服务器的编号(myid设置的值)；
>
>　　　　B：是这个服务器的 ip 地址；
>
>　　　　C：Leader选举的端口；
>
>　　　　D：Zookeeper服务器之间的通信端口。

### 3.2.3 配置系统环境变量

> 为了能够在任意目录启动zookeeper集群，我们需要配置环境变量

```linux
$ vim /etc/profile

#set zookeeper environment
export ZK_HOME=/data/apache-zookeeper-3.6.0-bin
export PATH=$PATH:$ZK_HOME/bin

$ source /etc/profile
```

### 3.2.4 启动zookeeper集群服务

```
$ zkServer.sh start         #启动服务

$ zkServer.sh stop          #停止服务

$ zkServer.sh restart       #重启服务

$ zkServer.sh status        #查看服务状态
```

>　我们分别对集群三台机器执行启动命令。执行完毕后，分别查看集群节点状态：
>
>　出现如下即是集群搭建成功：

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200309103415.png)

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200309103423.png)

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200309103433.png)

## 3.4 Zookeeper基础知识

### 3.4.1 Zookeeper存储结构

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200309123305.png)

### 3.4.2 Zookeeper分布式协调工具应用场景

* **命名服务(注册中心)Dubbo注册中心(动态实现负载均衡)**
  * 参考实现 zookeeper-loadbalance([github源码地址](https://github.com/FocusProgram/springcloud-zookeeper/tree/master/zookeeper-loadbalance)) 

* **分布式配置中心，动态管理配置文件**
 
* **消息中间件，事件通知(发布订阅)**
 
* **Zookeeper分布式事务(全局协调者)**
 
* **Zookeeper实现分布式锁**
    * 参考实现 zookeeper-lock([github源码地址](https://github.com/FocusProgram/springcloud-zookeeper/tree/master/zookeeper-lock))   
    
    > 分布式锁解决方案（目的：为了保证在分布式中共享数据安全问题）
    >
    > 1.数据库实现分布式锁（不推荐，效率特别低）
    >
    > 2.基于Redis实现分布式锁（考虑死锁、释放问题）redissession分布式锁
    >
    > 3.基于Zookeeper实现分布式锁（临时节点释放锁、失效时间容易控制）    

    > Zookeeper实现分布式锁的分类:
    >
    > 1.保持独占：所谓保持独占，就是所有试图来获取这个锁的客户端，最终只有一个可以成功获得这把锁。通常的做法是把 zk 上的一个 znode 看作是一把锁，通过 create znode 的方式来实现。所有客户端都去创建 /distribute_lock 节点，最终成功创建的那个客户端也即拥有了这把锁。
    >
    > 2.控制时序：就是所有视图来获取这个锁的客户端，最终都是会被安排执行，只是有个全局时序了。做法和上面基本类似，只是这里 /distributelock 已经预先存在，客户端在它下面创建临时有序节点（这个可以通过节点的属性控制：CreateMode.EPHEMERALSEQUENTIAL 来指定）。Zk 的父节点（/distribute_lock）维持一份 sequence, 保证子节点创建的时序性，从而也形成了每个客户端的全局时序。
    
    > Zookeeper实现原理：
    >
    > 多个JVM在同一个Zookeeper创建一个相同的临时节点，由于同级节点不允许重复特性保证只能有一个JVM创建节点
    
    > Zookeeper如何获取锁？
    >
    > 那个JVM创建节点快，就先拿到锁
    
    > Zookeeper如何释放锁？
    >
    > 拿到锁的JVM执行完程序，关闭当前的session会话，通过事件通知给其余的JVM进行重新等待抢锁

* **Zookeeper实现选举策略(哨兵机制)**
  * 参考实现 zookeeper-master([github源码地址](https://github.com/FocusProgram/springcloud-zookeeper/tree/master/zookeeper-master))  

* **Zookeeper实现本地负载均衡**

### 3.4.3 Zookeeper节点类型

* **持久节点**
    *  创建的节点永久持久化到硬盘上
* **临时节点**
    *  当前节点和会话链接保持，如果链接断开，则临时节点被删除