---
layout: post
title: zookeeper伪集群
category: zookeeper
tags: zk,集群
description: zk集群
--- 
#### 伪集群

zookeeper除了在单机模式下允许之外还有集群模式，即多个zk实例构成一个zk集群，如果条件有限比如只有一台机器，我们可以在一台机器上运行多个zk实例，这就是伪集群。


#### 源码

从github下载zookeeper的源码，复制出三个副本，准备启动三台zk实例，如下图。


![](http://7x00ae.com1.z0.glb.clouddn.com/18-2-13/31261271.jpg)


#### 配置文件

以下是zk的配置文件，伪集群的关键之处在于server.x=host:port:port 。
    
    # The number of milliseconds of each tick
    tickTime=2000
    # The number of ticks that the initial 
    # synchronization phase can take
    initLimit=10
    # The number of ticks that can pass between 
    # sending a request and getting an acknowledgement
    syncLimit=5
    # the directory where the snapshot is stored.
    # do not use /tmp for storage, /tmp here is just 
    # example sakes.
    dataDir=/zk/instance0/data
    # the port at which the clients will connect
    clientPort=2180
    # the maximum number of client connections.
    # increase this if you need to handle more clients
    #maxClientCnxns=60
    #
    # Be sure to read the maintenance section of the 
    # administrator guide before turning on autopurge.
    #
    # http://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_maintenance
    #
    # The number of snapshots to retain in dataDir
    #autopurge.snapRetainCount=3
    # Purge task interval in hours
    # Set to "0" to disable auto purge feature
    #autopurge.purgeInterval=1
    #关键的伪代集群信息
    server.0=127.0.0.1:2777:3777
    server.1=127.0.0.1:2888:3888
    server.2=127.0.0.1:2999:3999

##### server.1=127.0.0.1:2777:3777

1 表示该zk的编号为1，是zxid的一部分。

127.0.0.1 是表示zk实例所在机器的ip地址，端口2777表示 数据同步端口，而端口3777表示选举端口。



##### data目录

    dataDir=/zk/instance0/data
    
表示该zk实例的数据目录，在集群模式下，有一个myid文件，里面只有一个数字，比如0。表示当前zk实例 id=0.




#### 启动

第一次同时启动三台zk实例后，发现zk id最大的那个实例被选取称为了leader。

![](http://7x00ae.com1.z0.glb.clouddn.com/18-2-13/38434723.jpg)


当我们主动shutdown 实例2之后，可以发现实例1称为了leader。

主要原因是当每个实例的数据新鲜度一样，取实例id大的那个实例为leader。