# zookeeper入门与实战

## 1. zookeeper介绍

ZooKeeper是一个为分布式应用所设计的分布的、开源的协调服务，它主要是用来解决分布式应用中经常遇到的一些数据管理问题，如：统一命名服务、状 态同步服务、集群管理、分布式应用配置项的管理等，简化分布式应用协调及其管理的难度，提供高性能的分布式服务。Zookeeper的目标就是封装好复杂 易出错的关键服务，将简单易用的接口和性能高效、功能稳定的系统提供给用户。ZooKeeper本身可以以Standalone模式安装运行，不过它的长 处在于通过分布式ZooKeeper集群（一个Leader，多个Follower），基于一定的策略来保证ZooKeeper集群的稳定性和可用性，从 而实现分布式应用的可靠性。  

最新的版本可以在官网[http://zookeeper.apache.org/releases.html](http://zookeeper.apache.org/releases.html)来下载zookeeper的最新版本，我下载的是zookeeper-3.4.6.tar.gz，下面将从单机模式和集群模式两个方面介绍 Zookeeper 的安装和配置。

### 1.1 zookeeper的主要功能

组管理服务、分布式配置服务、分布式同步服务、分布式命名服务

### 1.2 Zookeeper的架构

### 1.3 Zookeeper的特点

| 特点    | 说明                                       |
| ----- | ---------------------------------------- |
| 最终一致性 | 为客户端展示同一个视图，这是zookeeper里面一个非常重要的功能       |
| 可靠性   | 如果消息被到一台服务器接受，那么它将被所有的服务器接受。             |
| 实时性   | Zookeeper不能保证两个客户端能同时得到刚更新的数据，如果需要最新数据，应该在读数据之前调用sync()接口。 |
| 独立性   | 各个Client之间互不干预                           |
| 原子性   | 更新只能成功或者失败，没有中间状态。                       |
| 顺序性   | 所有Server，同一消息发布顺序一致。                     |

### 1.4 zookeeper的工作原理

1.每个Server在内存中存储了一份数据； 
2.Zookeeper启动时，将从实例中选举一个leader（Paxos协议） 
3.Leader负责处理数据更新等操作（Zab协议）； 
4.一个更新操作成功，当且仅当大多数Server在内存中成功修改数据。

### 1.5 zookeeper中的几个重要角色

| 角色名           | 描述                                       |
| ------------- | ---------------------------------------- |
| 领导者(Leader)   | 领导者负责进行投票的发起和决议，更新系统状态，处理写请求             |
| 跟随者(Follwer)  | Follower用于接收客户端的读写请求并向客户端返回结果，在选主过程中参与投票 |
| 观察者（Observer） | 观察者可以接收客户端的读写请求，并将写请求转发给Leader，但Observer节点不参与投票过程，只同步leader状态，Observer的目的是为了，扩展系统，提高读取速度。在3.3.0版本之后，引入Observer角色的原因：Zookeeper需保证高可用和强一致性； 为了支持更多的客户端，需要增加更多Server； Server增多，投票阶段延迟增大，影响性能； 权衡伸缩性和高吞吐率，引入Observer Observer不参与投票； Observers接受客户端的连接，并将写请求转发给leader节点； 加入更多Observer节点，提高伸缩性，同时不影响吞吐率。 |
| 客户端(Client)   | 执行读写请求的发起方                               |

### 1.6 zookeeper集群的数目一般为奇数的原因

Leader选举算法采用了Paxos协议； 
Paxos核心思想：当多数Server写成功，则任务数据写成功 
如果有3个Server，则两个写成功即可； 
如果有4或5个Server，则三个写成功即可。 
Server数目一般为奇数（3、5、7） 
如果有3个Server，则最多允许1个Server挂掉； 
如果有4个Server，则同样最多允许1个Server挂掉 
由此，我们看出3台服务器和4台服务器的的容灾能力是一样的，所以 
为了节省服务器资源，一般我们采用奇数个数，作为服务器部署个数。

### 1.7 zookeeper的数据模型

基于树形结构的命名空间，与文件系统类似 
节点（znode）都可以存数据，可以有子节点 
节点不支持重命名 
数据大小不超过1MB（可配置） 
数据读写要保证完整性 
层次化的目录结构，命名符合常规文件系统规范； 
每个节点在zookeeper中叫做znode,并且其有一个唯一的路径标识； 
节点Znode可以包含数据和子节点（EPHEMERAL类型的节点不能有子节点）； 
Znode中的数据可以有多个版本，比如某一个路径下存有多个数据版本，那么查询这个路径下的数据需带上版本； 
客户端应用可以在节点上设置监视器（Watcher）； 
节点不支持部分读写，而是一次性完整读写。

Znode有两种类型，短暂的（ephemeral）和持久的（persistent）； 
Znode的类型在创建时确定并且之后不能再修改； 
短暂znode的客户端会话结束时，zookeeper会将该短暂znode删除，短暂znode不可以有子节点； 
持久znode不依赖于客户端会话，只有当客户端明确要删除该持久znode时才会被删除；
Znode有四种形式的目录节点，PERSISTENT、PERSISTENT_SEQUENTIAL、EPHEMERAL、EPHEMERAL_SEQUENTIAL。

### 1.8 zookeeper的主要应用场景

|          |                                          |                                          |
| -------- | ---------------------------------------- | ---------------------------------------- |
| 场景       | 介绍                                       | 实现思路                                     |
| 统一命名服务   | 分布式环境下，经常需要对应用/服务进行统一命名，便于识别不同服务； 类似于域名与ip之间对应关系，域名容易记住； 通过名称来获取资源或服务的地址，提供者等信息 按照层次结构组织服务/应用名称 可将服务名称以及地址信息写到Zookeeper上，客户端通过Zookeeper获取可用服务列表类 |                                          |
| 配置管理服务   | 分布式环境下，配置文件管理和同步是一个常见问题； 一个集群中，所有节点的配置信息是一致的，比如Hadoop； 对配置文件修改后，希望能够快速同步到各个节点上 配置管理可交由Zookeeper实现； 可将配置信息写入Zookeeper的一个znode上； 各个节点监听这个znode 一旦znode中的数据被修改，zookeeper将通知各个节点 |                                          |
| 集群管理     | 分布式环境中，实时掌握每个节点的状态是必要的； 可根据节点实时状态作出一些调整； 可交由Zookeeper实现； 可将节点信息写入Zookeeper的一个znode上； 监听这个znode可获取它的实时状态变化 典型应用 Hbase中Master状态监控与选举 | 在 Zookeeper 上创建一个 EPHEMERAL 类型的目录节点，然后每个 Server 在它们创建目录节点的父目录节点上调用[getChildren](http://hadoop.apache.org/zookeeper/docs/r3.2.2/api/org/apache/zookeeper/ZooKeeper.html#getChildren%28java.lang.String,%20boolean%29)([String](http://java.sun.com/javase/6/docs/api/java/lang/String.html?is-external=true) path, boolean watch) 方法并设置 watch 为 true，由于是 EPHEMERAL 目录节点，当创建它的 Server 死去，这个目录节点也随之被删除，所以 Children 将会变化，这时 [getChildren](http://hadoop.apache.org/zookeeper/docs/r3.2.2/api/org/apache/zookeeper/ZooKeeper.html#getChildren%28java.lang.String,%20boolean%29)上的 Watch 将会被调用，所以其它 Server 就知道已经有某台 Server 死去了。新增 Server 也是同样的原理。 |
| 分布式通知和协调 | 分布式环境中，经常存在一个服务需要知道它所管理的子服务的状态； NameNode须知道各DataNode的状态 JobTracker须知道各TaskTracker的状态 心跳检测机制可通过Zookeeper实现； 信息推送可由Zookeeper实现（发布/订阅模式） |                                          |
| 分布式锁     | Zookeeper是强一致的； 多个客户端同时在Zookeeper上创建相同znode，只有一个创建成功。 实现锁的独占性 多个客户端同时在Zookeeper上创建相同znode ，创建成功的那个客户端得到锁，其他客户端等待。 控制锁的时序 各个客户端在某个znode下创建临时znode （类型为CreateMode.EPHEMERAL_SEQUENTIAL），这样，该znode可掌握全局访问时序。 | 实现方式是需要获得锁的 Server 创建一个 EPHEMERAL_SEQUENTIAL 目录节点，然后调用 [getChildren](http://hadoop.apache.org/zookeeper/docs/r3.2.2/api/org/apache/zookeeper/ZooKeeper.html#getChildren%28java.lang.String,%20boolean%29)方法获取当前的目录节点列表中最小的目录节点是不是就是自己创建的目录节点，如果正是自己创建的，那么它就获得了这个锁，如果不是那么它就调用 [exists](http://hadoop.apache.org/zookeeper/docs/r3.2.2/api/org/apache/zookeeper/ZooKeeper.html#exists%28java.lang.String,%20boolean%29)([String](http://java.sun.com/javase/6/docs/api/java/lang/String.html?is-external=true) path, boolean watch) 方法并监控 Zookeeper 上目录节点列表的变化，一直到自己创建的节点是列表中最小编号的目录节点，从而获得锁，释放锁很简单，只要删除前面它自己所创建的目录节点就行了。 |
| 分布式队列    | 两种队列； 当一个队列的成员都聚齐时，这个队列才可用，否则一直等待所有成员到达，这种是同步队列。 队列按照 FIFO 方式进行入队和出队操作，例如实现生产者和消费者模型。（可通过分布式锁实现） 同步队列 一个job由多个task组成，只有所有任务完成后，job才运行完成。 可为job创建一个/job目录，然后在该目录下，为每个完成的task创建一个临时znode，一旦临时节点数目达到task总数，则job运行完成。 | 同步队列实现思路： 创建一个父目录 /synchronizing，每个成员都监控标志（Set Watch）位目录 /synchronizing/start 是否存在，然后每个成员都加入这个队列，加入队列的方式就是创建 /synchronizing/member_i 的临时目录节点，然后每个成员获取 / synchronizing 目录的所有目录节点，也就是 member_i。判断 i 的值是否已经是成员的个数，如果小于成员个数等待 /synchronizing/start 的出现，如果已经相等就创建 /synchronizing/start。FIFO 队列实现思路：实现的思路也非常简单，就是在特定的目录下创建 SEQUENTIAL 类型的子目录 /queue_i，这样就能保证所有成员加入队列时都是有编号的，出队列时通过 getChildren( ) 方法可以返回当前所有的队列中的元素，然后消费其中最小的一个，这样就能保证 FIFO。 |

## 2. zookeeper单节点(Standalone)模式

```java
sunguoli@sunguolideMacBook-Pro:~/zookeeper$ tar -zxvf zookeeper-3.4.6.tar.gz 
sunguoli@sunguolideMacBook-Pro:~/zookeeper$ cd zookeeper-3.4.6
sunguoli@sunguolideMacBook-Pro:~/zookeeper/zookeeper-3.4.6$ ls -al
total 3088
drwxr-xr-x@ 23 sunguoli  staff      782  9 28 15:15 .
drwxr-xr-x   9 sunguoli  staff      306  9 28 15:04 ..
-rw-r--r--@  1 sunguoli  staff    80776  2 20  2014 CHANGES.txt
-rw-r--r--@  1 sunguoli  staff    11358  2 20  2014 LICENSE.txt
-rw-r--r--@  1 sunguoli  staff      170  2 20  2014 NOTICE.txt
-rw-r--r--@  1 sunguoli  staff     1585  2 20  2014 README.txt
-rw-r--r--@  1 sunguoli  staff     1770  2 20  2014 README_packaging.txt
drwxr-xr-x@ 11 sunguoli  staff      374  9 29 15:50 bin
-rw-r--r--@  1 sunguoli  staff    82446  2 20  2014 build.xml
drwxr-xr-x@  9 sunguoli  staff      306  9 29 15:48 conf
drwxr-xr-x@ 10 sunguoli  staff      340  2 20  2014 contrib
drwxr-xr-x@ 22 sunguoli  staff      748  2 20  2014 dist-maven
drwxr-xr-x@ 49 sunguoli  staff     1666  2 20  2014 docs
-rw-r--r--@  1 sunguoli  staff     3375  2 20  2014 ivy.xml
-rw-r--r--@  1 sunguoli  staff     1953  2 20  2014 ivysettings.xml
drwxr-xr-x@ 11 sunguoli  staff      374  2 20  2014 lib
drwxr-xr-x@  5 sunguoli  staff      170  2 20  2014 recipes
drwxr-xr-x@ 11 sunguoli  staff      374  2 20  2014 src
-rw-r--r--@  1 sunguoli  staff  1340305  2 20  2014 zookeeper-3.4.6.jar
-rw-r--r--@  1 sunguoli  staff      836  2 20  2014 zookeeper-3.4.6.jar.asc
-rw-r--r--@  1 sunguoli  staff       33  2 20  2014 zookeeper-3.4.6.jar.md5
-rw-r--r--@  1 sunguoli  staff       41  2 20  2014 zookeeper-3.4.6.jar.sha1
-rw-r--r--   1 sunguoli  staff    23359  9 29 18:28 zookeeper.out
```

### **2.1 修改配置文件**

我要把zookeeper的数据放在/Users/sunguoli/zookeeper/data这个文件夹里面，另外zookeeper默认使用的配置文件时zoo.cfg，所以要把zoo_sample.cfg cp到zoo.cfg

```java
sunguoli@sunguolideMacBook-Pro:~/zookeeper/zookeeper-3.4.6$ mkdir /Users/sunguoli/zookeeper/data
sunguoli@sunguolideMacBook-Pro:~/zookeeper/zookeeper-3.4.6$ cp conf/zoo_sample.cfg conf/zoo.cfg
sunguoli@sunguolideMacBook-Pro:~/zookeeper/zookeeper-3.4.6$ vi conf/zoo.cfg
  1 # The number of milliseconds of each tick

  2 tickTime=2000

  3 # The number of ticks that the initial 

  4 # synchronization phase can take

  5 initLimit=10

  6 # The number of ticks that can pass between 

  7 # sending a request and getting an acknowledgement

  8 syncLimit=5

  9 # the directory where the snapshot is stored.

 10 # do not use /tmp for storage, /tmp here is just 

 11 # example sakes.

 12 dataDir=/Users/sunguoli/zookeeper/data

 13 # the port at which the clients will connect

 14 clientPort=2181

 15 # the maximum number of client connections.

 16 # increase this if you need to handle more clients

 17 #maxClientCnxns=60

 18 #

 19 # Be sure to read the maintenance section of the 

 20 # administrator guide before turning on autopurge.

 21 #

 22 # http://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_maintenance

 23 #

 24 # The number of snapshots to retain in dataDir

 25 #autopurge.snapRetainCount=3

 26 # Purge task interval in hours

 27 # Set to "0" to disable auto purge feature

 28 #autopurge.purgeInterval=1
```

各个配置项的解释：

1. tickTime：这个时间是作为 Zookeeper 服务器之间或客户端与服务器之间维持心跳的时间间隔，也就是每个 tickTime 时间就会发送一个心跳。
2. initLimit：这个配置项是用来配置 Zookeeper 接受客户端（这里所说的客户端不是用户连接 Zookeeper 服务器的客户端，而是 Zookeeper 服务器集群中连接到 Leader 的 Follower 服务器）初始化连接时最长能忍受多少个心跳时间间隔数。当已经超过 10 个心跳的时间（也就是 tickTime）长度后 Zookeeper 服务器还没有收到客户端的返回信息，那么表明这个客户端连接失败。总的时间长度就是 5*2000=10 秒
3. syncLimit：这个配置项标识 Leader 与 Follower 之间发送消息，请求和应答时间长度，最长不能超过多少个 tickTime 的时间长度，总的时间长度就是 2*2000=4 秒
4. dataDir：顾名思义就是 Zookeeper 保存数据的目录，默认情况下，Zookeeper 将写数据的日志文件也保存在这个目录里。我们上文配置的目录是：/Users/sunguoli/zookeeper/data
5. clientPort：这个端口就是客户端连接 Zookeeper 服务器的端口，Zookeeper 会监听这个端口，接受客户端的访问请求。

### **2.2 启动zookeeper**

```java
sunguoli@sunguolideMacBook-Pro:~/zookeeper/zookeeper-3.4.6$ bin/zkServer.sh start
JMX enabled by default
Using config: /Users/sunguoli/zookeeper/zookeeper-3.4.6/bin/../conf/zoo.cfg
Starting zookeeper ... STARTED
sunguoli@sunguolideMacBook-Pro:~/zookeeper/zookeeper-3.4.6$ zkServer.sh status
JMX enabled by default
Using config: /Users/sunguoli/zookeeper/zookeeper-3.4.6/bin/../conf/zoo.cfg
Mode: standalone
```

从命令行中看到，zookeeper已经启动了，模式是standalone，使用的配置文件是zoo.cfg

### 2.3 连接zookeeper

```java
sunguoli@sunguolideMacBook-Pro:~/zookeeper/zookeeper-3.4.6$ bin/zkCli.sh
Connecting to localhost:2181
...
Welcome to ZooKeeper!
...
WATCHER::
WatchedEvent state:SyncConnected type:None path:null
#用man或者help查看下有哪些可用的命令，我们看到了很多熟悉的命令
[zk: localhost:2181(CONNECTED) 0] man
ZooKeeper -server host:port cmd args
	connect host:port
	get path [watch]
	ls path [watch]
	set path data [version]
	rmr path
	delquota [-n|-b] path
	quit 
	printwatches on|off
	create [-s] [-e] path data acl
	stat path [watch]
	close 
	ls2 path [watch]
	history 
	listquota path
	setAcl path acl
	getAcl path
	sync path
	redo cmdno
	addauth scheme auth
	delete path [version]
	setquota -n|-b val path
[zk: localhost:2181(CONNECTED) 1] help
ZooKeeper -server host:port cmd args
	connect host:port
	get path [watch]
	ls path [watch]
	set path data [version]
	rmr path
	delquota [-n|-b] path
	quit 
	printwatches on|off
	create [-s] [-e] path data acl
	stat path [watch]
	close 
	ls2 path [watch]
	history 
	listquota path
	setAcl path acl
	getAcl path
	sync path
	redo cmdno
	addauth scheme auth
	delete path [version]
	setquota -n|-b val path
#ZooKeeper的结构，很像是目录结构，ls一下，看到了一个默认的节点zookeeper
[zk: localhost:2181(CONNECTED) 0] ls /
[zookeeper]
#创建一个新的节点，/node, 值是helloword
[zk: localhost:2181(CONNECTED) 1] create /node helloword
Created /node
#再看一下，恩，多了一个我们新建的节点/node,和zookeeper都是在/根目录下的。
[zk: localhost:2181(CONNECTED) 2] ls /
[node, zookeeper]
#看看节点的值是啥？还真是我们设置的helloword,还显示了创建时间，修改时间，version,长度，children个数等。
[zk: localhost:2181(CONNECTED) 3] get /node
helloword
cZxid = 0x32
ctime = Wed Sep 30 15:06:10 CST 2015
mZxid = 0x32
mtime = Wed Sep 30 15:06:10 CST 2015
pZxid = 0x32
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 9
numChildren = 0
#修改值，看看，创时间没变，修改时间变了，长度变了，数据版本值变了。
[zk: localhost:2181(CONNECTED) 4] set /node helloword!
cZxid = 0x32
ctime = Wed Sep 30 15:06:10 CST 2015
mZxid = 0x33
mtime = Wed Sep 30 15:06:55 CST 2015
pZxid = 0x32
cversion = 0
dataVersion = 1
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 10
numChildren = 0
#再看看
[zk: localhost:2181(CONNECTED) 5] get /node
helloword!
cZxid = 0x32
ctime = Wed Sep 30 15:06:10 CST 2015
mZxid = 0x33
mtime = Wed Sep 30 15:06:55 CST 2015
pZxid = 0x32
cversion = 0
dataVersion = 1
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 10
numChildren = 0
#给他创建一个子节点，值testchild
[zk: localhost:2181(CONNECTED) 6] create /node/childnode testchild
Created /node/childnode
#看看根目录，恩，没变
[zk: localhost:2181(CONNECTED) 7] ls /
[node, zookeeper]
#命令输错鸟。。。⊙﹏⊙b汗
[zk: localhost:2181(CONNECTED) 8] ls /node/
Command failed: java.lang.IllegalArgumentException: Path must not end with / character
#看一看
[zk: localhost:2181(CONNECTED) 9] ls /node 
[childnode]
#childdren的数目变了
[zk: localhost:2181(CONNECTED) 10] get /node
helloword!
cZxid = 0x32
ctime = Wed Sep 30 15:06:10 CST 2015
mZxid = 0x33
mtime = Wed Sep 30 15:06:55 CST 2015
pZxid = 0x34
cversion = 1
dataVersion = 1
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 10
numChildren = 1

#再建一个根节点
[zk: localhost:2181(CONNECTED) 11] create /test test
Created /test
#多了个根节点哦~
[zk: localhost:2181(CONNECTED) 12] ls /
[node, test, zookeeper]
#删
[zk: localhost:2181(CONNECTED) 13] delete /test
#偶，不见了
[zk: localhost:2181(CONNECTED) 14] ls /
[node, zookeeper]
#看看默认的节点有啥？
[zk: localhost:2181(CONNECTED) 15] ls /zookeeper
[quota]
[zk: localhost:2181(CONNECTED) 16] get /zookeeper
cZxid = 0x0
ctime = Thu Jan 01 08:00:00 CST 1970
mZxid = 0x0
mtime = Thu Jan 01 08:00:00 CST 1970
pZxid = 0x0
cversion = -1
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 0
numChildren = 1
[zk: localhost:2181(CONNECTED) 17] get /zookeeper/quota
cZxid = 0x0
ctime = Thu Jan 01 08:00:00 CST 1970
mZxid = 0x0
mtime = Thu Jan 01 08:00:00 CST 1970
pZxid = 0x0
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 0
numChildren = 0
#退出
[zk: localhost:2181(CONNECTED) 18] quit
Quitting...
2015-09-30 15:08:40,639 [myid:] - INFO  [main:ZooKeeper@684] - Session: 0x1501d03ae510003 closed
2015-09-30 15:08:40,639 [myid:] - INFO  [main-EventThread:ClientCnxn$EventThread@512] - EventThread shut down
sunguoli@sunguolideMacBook-Pro:~/zookeeper/zookeeper-3.4.6$
```

### 2.4 stop zookeeper

```java
sunguoli@sunguolideMacBook-Pro:~/zookeeper/zookeeper-3.4.6$ bin/zkServer.sh stop
JMX enabled by default
Using config: /Users/sunguoli/zookeeper/zookeeper-3.4.6/bin/../conf/zoo.cfg
Stopping zookeeper ... STOPPED
sunguoli@sunguolideMacBook-Pro:~/zookeeper/zookeeper-3.4.6$
```

## **3. zookeeper伪分布式集群模式**

上面体验了单节点模式，我们来体验下伪分布式模式，在一台机器上跑多个zookeeper节点。当然伪分布式模式是区别于完全分布式模式有多台机器，每天机器上一个zookeeper实例。

### 3.1 配置zookeeper

我们来建一个3个节点的zookeeper伪分布式集群

```java
#建3个数据目录
sunguoli@sunguolideMacBook-Pro:~/zookeeper$ mkdir /Users/sunguoli/zookeeper/data1
sunguoli@sunguolideMacBook-Pro:~/zookeeper$ mkdir /Users/sunguoli/zookeeper/data2
sunguoli@sunguolideMacBook-Pro:~/zookeeper$ mkdir /Users/sunguoli/zookeeper/data3


#分别新建myid文件
sunguoli@sunguolideMacBook-Pro:~/zookeeper$ echo "1" > data1/myid
sunguoli@sunguolideMacBook-Pro:~/zookeeper$ echo "2" > data2/myid
sunguoli@sunguolideMacBook-Pro:~/zookeeper$ echo "3" > data3/myid
 
#修改配置文件，主要是dataDir, clientPort, server
#zk1
sunguoli@sunguolideMacBook-Pro:~/zookeeper/zookeeper-3.4.6$ cat conf/zk1.cfg 
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
dataDir=/Users/sunguoli/zookeeper/data1
# the port at which the clients will connect
clientPort=2181
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
server.1=127.0.0.1:2888:3888
server.2=127.0.0.1:2889:3889
server.3=127.0.0.1:2890:3890


#zk2
sunguoli@sunguolideMacBook-Pro:~/zookeeper/zookeeper-3.4.6$ cat conf/zk2.cfg 
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
dataDir=/Users/sunguoli/zookeeper/data2
# the port at which the clients will connect
clientPort=2182
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
server.1=127.0.0.1:2888:3888
server.2=127.0.0.1:2889:3889
server.3=127.0.0.1:2890:3890


#zk3
sunguoli@sunguolideMacBook-Pro:~/zookeeper/zookeeper-3.4.6$ cat conf/zk3.cfg 
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
dataDir=/Users/sunguoli/zookeeper/data3
# the port at which the clients will connect
clientPort=2183
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
server.1=127.0.0.1:2888:3888
server.2=127.0.0.1:2889:3889
server.3=127.0.0.1:2890:3890
```

其中注意：

- server.A=B：C：D：其中 A 是一个数字，表示这个是第几号服务器；B 是这个服务器的 ip 地址；C 表示的是这个服务器与集群中的 Leader 服务器交换信息的端口；D 表示的是万一集群中的 Leader 服务器挂了，需要一个端口来重新进行选举，选出一个新的 Leader，而这个端口就是用来执行选举时服务器相互通信的端口。如果是伪集群的配置方式，由于 B 都是一样，所以不同的 Zookeeper 实例通信端口号不能一样，所以要给它们分配不同的端口号。
- 除了修改 zoo.cfg 配置文件，集群模式下还要配置一个文件 myid，这个文件在 dataDir 目录下，这个文件里面就有一个数据就是 A 的值，Zookeeper 启动时会读取这个文件，拿到里面的数据与 zoo.cfg 里面的配置信息比较从而判断到底是那个 server。

因为这三个节点是在同一台机器上，所以分别配置了不同的端口，不同的数据目录。

### 3.2 启动集群

集群模式的zk,每个节点一次启动。

```java
#start
sunguoli@sunguolideMacBook-Pro:~/zookeeper/zookeeper-3.4.6$ bin/zkServer.sh start conf/zk1.cfg 
JMX enabled by default
Using config: conf/zk1.cfg
Starting zookeeper ... STARTED
sunguoli@sunguolideMacBook-Pro:~/zookeeper/zookeeper-3.4.6$ bin/zkServer.sh start conf/zk2.cfg 
JMX enabled by default
Using config: conf/zk2.cfg
Starting zookeeper ... STARTED
sunguoli@sunguolideMacBook-Pro:~/zookeeper/zookeeper-3.4.6$ bin/zkServer.sh start conf/zk3.cfg 
JMX enabled by default
Using config: conf/zk3.cfg
Starting zookeeper ... STARTED
 
#status
sunguoli@sunguolideMacBook-Pro:~/zookeeper/zookeeper-3.4.6$ bin/zkServer.sh status conf/zk1.cfg 
JMX enabled by default
Using config: conf/zk1.cfg
Mode: follower
sunguoli@sunguolideMacBook-Pro:~/zookeeper/zookeeper-3.4.6$ bin/zkServer.sh status conf/zk2.cfg 
JMX enabled by default
Using config: conf/zk2.cfg
Mode: leader
sunguoli@sunguolideMacBook-Pro:~/zookeeper/zookeeper-3.4.6$ bin/zkServer.sh status conf/zk3.cfg 
JMX enabled by default
Using config: conf/zk3.cfg
Mode: follower
```

从3个节点的状态可以看出，zk2是leader, zk1和zk3是follower.

### 3.3 连接zk集群

我启动了三个terminal来分别连接集群

```java
sunguoli@sunguolideMacBook-Pro:~/zookeeper/zookeeper-3.4.6$ bin/zkCli.sh -server localhost:2181
Connecting to localhost:2181
...
Welcome to ZooKeeper!
...
[zk: localhost:2181(CONNECTED) 0]
 
sunguoli@sunguolideMacBook-Pro:~/zookeeper/zookeeper-3.4.6$ bin/zkCli.sh -server localhost:2182
Connecting to localhost:2182
...
Welcome to ZooKeeper!
...
[zk: localhost:2182(CONNECTED) 0]
 
sunguoli@sunguolideMacBook-Pro:~/zookeeper/zookeeper-3.4.6$ bin/zkCli.sh -server localhost:2183
Connecting to localhost:2183
...
Welcome to ZooKeeper!
...
[zk: localhost:2183(CONNECTED) 0]
```

### 3.4 测试zk集群

```java
#在zk1上新建节点
[zk: localhost:2181(CONNECTED) 0] ls /
[zookeeper]
[zk: localhost:2181(CONNECTED) 7] create /node node_in_root
Created /node
[zk: localhost:2181(CONNECTED) 9] ls /
[node, zookeeper]
[zk: localhost:2181(CONNECTED) 10] get /node
node_in_root
cZxid = 0x20000000a
ctime = Wed Sep 30 16:24:00 CST 2015
mZxid = 0x20000000a
mtime = Wed Sep 30 16:24:00 CST 2015
pZxid = 0x20000000a
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 12
numChildren = 0
 
#zk2上来看看，我们在zk1上新建的节点从zk2上也能看到
[zk: localhost:2182(CONNECTED) 1] ls /
[node, zookeeper]
[zk: localhost:2182(CONNECTED) 2] get /node
node_in_root
cZxid = 0x20000000a
ctime = Wed Sep 30 16:24:00 CST 2015
mZxid = 0x20000000a
mtime = Wed Sep 30 16:24:00 CST 2015
pZxid = 0x20000000a
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 12
numChildren = 0


#zk3上来看看，然后修改下试试
[zk: localhost:2183(CONNECTED) 0] get /node
node_in_root
cZxid = 0x20000000a
ctime = Wed Sep 30 16:24:00 CST 2015
mZxid = 0x20000000a
mtime = Wed Sep 30 16:24:00 CST 2015
pZxid = 0x20000000a
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 12
numChildren = 0
[zk: localhost:2183(CONNECTED) 1] set /node node_in_root_modified_by_3
cZxid = 0x20000000a
ctime = Wed Sep 30 16:24:00 CST 2015
mZxid = 0x20000000b
mtime = Wed Sep 30 16:25:24 CST 2015
pZxid = 0x20000000a
cversion = 0
dataVersion = 1
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 26
numChildren = 0
[zk: localhost:2183(CONNECTED) 2] get /node                           
node_in_root_modified_by_3
cZxid = 0x20000000a
ctime = Wed Sep 30 16:24:00 CST 2015
mZxid = 0x20000000b
mtime = Wed Sep 30 16:25:24 CST 2015
pZxid = 0x20000000a
cversion = 0
dataVersion = 1
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 26
numChildren = 0
 
#zk2上看到，节点的值变了哎
[zk: localhost:2182(CONNECTED) 3] get /node
node_in_root_modified_by_3
cZxid = 0x20000000a
ctime = Wed Sep 30 16:24:00 CST 2015
mZxid = 0x20000000b
mtime = Wed Sep 30 16:25:24 CST 2015
pZxid = 0x20000000a
cversion = 0
dataVersion = 1
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 26
numChildren = 0
 
#zk1上看到的值也变了哎
[zk: localhost:2181(CONNECTED) 11] get /node
node_in_root_modified_by_3
cZxid = 0x20000000a
ctime = Wed Sep 30 16:24:00 CST 2015
mZxid = 0x20000000b
mtime = Wed Sep 30 16:25:24 CST 2015
pZxid = 0x20000000a
cversion = 0
dataVersion = 1
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 26
numChildren = 0
 
#在zk2上建个子节点
[zk: localhost:2182(CONNECTED) 4] create /node/childnode secend_node_create_in_2
Created /node/childnode
 
#zk1上看到子节点了
[zk: localhost:2181(CONNECTED) 12] get /node
node_in_root_modified_by_3
cZxid = 0x20000000a
ctime = Wed Sep 30 16:24:00 CST 2015
mZxid = 0x20000000b
mtime = Wed Sep 30 16:25:24 CST 2015
pZxid = 0x20000000c
cversion = 1
dataVersion = 1
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 26
numChildren = 1
[zk: localhost:2181(CONNECTED) 13] ls /node
[childnode]
[zk: localhost:2181(CONNECTED) 14] ls /node/childnode
[]
[zk: localhost:2181(CONNECTED) 15] get /node/childnode
secend_node_create_in_2
cZxid = 0x20000000c
ctime = Wed Sep 30 16:26:41 CST 2015
mZxid = 0x20000000c
mtime = Wed Sep 30 16:26:41 CST 2015
pZxid = 0x20000000c
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 23
numChildren = 0


#zk3上也看到子节点了呢
[zk: localhost:2183(CONNECTED) 3] get /node/childnode
secend_node_create_in_2
cZxid = 0x20000000c
ctime = Wed Sep 30 16:26:41 CST 2015
mZxid = 0x20000000c
mtime = Wed Sep 30 16:26:41 CST 2015
pZxid = 0x20000000c
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 23
numChildren = 0
 
#zk3上删除子节点
[zk: localhost:2183(CONNECTED) 4] ls /
[node, zookeeper]
[zk: localhost:2183(CONNECTED) 5] delete /node
Node not empty: /node
[zk: localhost:2183(CONNECTED) 6] delete /node/childnode
[zk: localhost:2183(CONNECTED) 7] ls /node
[]
[zk: localhost:2183(CONNECTED) 8] ls /
[node, zookeeper]
 
#zk2上看到子节点也被删除了，再删除/node
[zk: localhost:2182(CONNECTED) 5] ls /
[node, zookeeper]
[zk: localhost:2182(CONNECTED) 6] get /node
node_in_root_modified_by_3
cZxid = 0x20000000a
ctime = Wed Sep 30 16:24:00 CST 2015
mZxid = 0x20000000b
mtime = Wed Sep 30 16:25:24 CST 2015
pZxid = 0x20000000e
cversion = 2
dataVersion = 1
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 26
numChildren = 0
[zk: localhost:2182(CONNECTED) 7] delete /node
[zk: localhost:2182(CONNECTED) 8] ls /
[zookeeper]


#zk1上看到节点也咩有了
[zk: localhost:2181(CONNECTED) 16] ls /
[zookeeper]
```

### 3.4 stop zk集群

我们来看看如何stop集群

```java
#先看一眼各个zk节点的状态
sunguoli@sunguolideMacBook-Pro:~/zookeeper/zookeeper-3.4.6$ bin/zkServer.sh status conf/zk1.cfg 
JMX enabled by default
Using config: conf/zk1.cfg
Mode: follower
sunguoli@sunguolideMacBook-Pro:~/zookeeper/zookeeper-3.4.6$ bin/zkServer.sh status conf/zk2.cfg 
JMX enabled by default
Using config: conf/zk2.cfg
Mode: leader
sunguoli@sunguolideMacBook-Pro:~/zookeeper/zookeeper-3.4.6$ bin/zkServer.sh status conf/zk3.cfg 
JMX enabled by default
Using config: conf/zk3.cfg
Mode: follower
 
#开始stop节点了啊，先从哪个下手呢？恩，先从leader下手。可以看到，zk2 stopped以后，zk3成了leader,zk1还是follower
sunguoli@sunguolideMacBook-Pro:~/zookeeper/zookeeper-3.4.6$ bin/zkServer.sh stop conf/zk2.cfg 
JMX enabled by default
Using config: conf/zk2.cfg
Stopping zookeeper ... STOPPED
sunguoli@sunguolideMacBook-Pro:~/zookeeper/zookeeper-3.4.6$ bin/zkServer.sh status conf/zk1.cfg 
JMX enabled by default
Using config: conf/zk1.cfg
Mode: follower
sunguoli@sunguolideMacBook-Pro:~/zookeeper/zookeeper-3.4.6$ bin/zkServer.sh status conf/zk3.cfg 
JMX enabled by default
Using config: conf/zk3.cfg
Mode: leader
 
#还有两个节点，我们这次从zk1下手，zk1 stopped以后，zk3也罢工了。。。
sunguoli@sunguolideMacBook-Pro:~/zookeeper/zookeeper-3.4.6$ bin/zkServer.sh stop conf/zk1.cfg 
JMX enabled by default
Using config: conf/zk1.cfg
Stopping zookeeper ... STOPPED
sunguoli@sunguolideMacBook-Pro:~/zookeeper/zookeeper-3.4.6$ bin/zkServer.sh status conf/zk3.cfg 
JMX enabled by default
Using config: conf/zk3.cfg
Error contacting service. It is probably not running.
sunguoli@sunguolideMacBook-Pro:~/zookeeper/zookeeper-3.4.6$ bin/zkServer.sh stop conf/zk3.cfg 
JMX enabled by default
Using config: conf/zk3.cfg
Stopping zookeeper ... STOPPED
 
#两个节点时，一个leader,一个follower，follower stopped以后，leader罢工了，那么如果先停leader呢？
sunguoli@sunguolideMacBook-Pro:~/zookeeper/zookeeper-3.4.6$ bin/zkServer.sh start conf/zk1.cfg 
JMX enabled by default
Using config: conf/zk1.cfg
Starting zookeeper ... STARTED
sunguoli@sunguolideMacBook-Pro:~/zookeeper/zookeeper-3.4.6$ bin/zkServer.sh status conf/zk1.cfg 
JMX enabled by default
Using config: conf/zk1.cfg
Error contacting service. It is probably not running.
sunguoli@sunguolideMacBook-Pro:~/zookeeper/zookeeper-3.4.6$ bin/zkServer.sh start conf/zk2.cfg 
JMX enabled by default
Using config: conf/zk2.cfg
Starting zookeeper ... STARTED
sunguoli@sunguolideMacBook-Pro:~/zookeeper/zookeeper-3.4.6$ bin/zkServer.sh status conf/zk1.cfg 
JMX enabled by default
Using config: conf/zk1.cfg
Mode: leader
sunguoli@sunguolideMacBook-Pro:~/zookeeper/zookeeper-3.4.6$ bin/zkServer.sh status conf/zk2.cfg 
JMX enabled by default
Using config: conf/zk2.cfg
Mode: follower
sunguoli@sunguolideMacBook-Pro:~/zookeeper/zookeeper-3.4.6$ bin/zkServer.sh stop conf/zk1.cfg 
JMX enabled by default
Using config: conf/zk1.cfg
Stopping zookeeper ... STOPPED
sunguoli@sunguolideMacBook-Pro:~/zookeeper/zookeeper-3.4.6$ bin/zkServer.sh status conf/zk2.cfg 
JMX enabled by default
Using config: conf/zk2.cfg
Error contacting service. It is probably not running.
sunguoli@sunguolideMacBook-Pro:~/zookeeper/zookeeper-3.4.6$ bin/zkServer.sh stop conf/zk2.cfg 
JMX enabled by default
Using config: conf/zk2.cfg
Stopping zookeeper ... STOPPED
```

## 4. 用zookeeper进行配置管理

### 4.1 监听节点

```java
import org.apache.zookeeper.KeeperException;
import org.apache.zookeeper.WatchedEvent;
import org.apache.zookeeper.Watcher;
import org.apache.zookeeper.ZooKeeper;
import org.slf4j.LoggerFactory;
import org.slf4j.Logger;

import java.io.IOException;

/**
 * Desc:监控各个node的情况，然后写到数据库里面
 * Author:sunguoli
 * Date:15/9/25
 * Time:上午11:35
 */
public class zkWatcherClient implements Runnable, Watcher{
    private static final Logger logger = LoggerFactory.getLogger(zkWatcherClient.class);

    private String zkServer;
    private String watchedNode;
    //private int zkTimeout;
    private int zkWatchInterval;
    private ZooKeeper zooKeeper;

    public zkWatcherClient(String zkServer, String watchedNode, int zkTimeout, int interval) throws IOException{
        this.zkServer = zkServer;
        this.watchedNode = watchedNode;
        //this.zkTimeout = zkTimeout;
        this.zkWatchInterval = interval;
        this.zooKeeper = new ZooKeeper(zkServer, zkTimeout, this);
    }

    public void init() {
        try {
            Thread thread = new Thread(this);
            thread.start();
        } catch (Exception e) {
            logger.error("init zkWatcherClient, error:", e);
        }
    }

    @Override public void process(WatchedEvent event) {
        try{
            String nodeValue = new String(zooKeeper.getData(watchedNode, true, null));
            logger.info("========the latest zookeeper node status is:" + nodeValue);
            updateNodeStatusToDB(nodeValue);
        } catch (KeeperException e){
            e.printStackTrace();
        } catch (InterruptedException e){
            e.printStackTrace();
        }
    }

    @Override public void run() {
        while(true){
            try{
                logger.info("========web-zookeeper-watcher is watching...");
                zooKeeper.exists(watchedNode, this);
            }catch(KeeperException e){
                e.printStackTrace();
            }catch(InterruptedException e){
                e.printStackTrace();
            }

            try{
                logger.info("========web-zookeeper-watcher sleep...");
                Thread.sleep(zkWatchInterval);
            }catch(InterruptedException e){
                e.printStackTrace();
            }
        }
    }

    /**
     * 检查状态并写DB
     * @param nodeValue
     */
    public void updateNodeStatusToDB(String nodeValue){
        //TODO 检查是否在数据库中已经存在，不要写重复数据，如果不存在就写数据库
        //TODO node节点的格式是什么样的？
        logger.info("========writing DB:" + nodeValue);
    }

    public static void main(String[] args) throws Exception{
        zkWatcherClient client = new zkWatcherClient("localhost:2181", "/node", 2000, 2000);
        Thread thread = new Thread(client);
        thread.start();
    }

}
```

可以在配置信息写到配置文件中：

```java
#zookeeper properties
zookeeper_server=localhost:2181
#需要监控的节点
zookeeper_watch_node=/node
zookeeper_time_out=2000
zookeeper_watch_interval=2000
```

并且设置成随项目启动：

```java
<bean id="zkWatcherClient" class="com.*.*.*.zkWatcherClient"
        init-method="init">
    <constructor-arg index="0" value="${zookeeper_server}"/>
    <constructor-arg index="1" value="${zookeeper_watch_node}"/>
    <constructor-arg index="2" value="${zookeeper_time_out}"/>
    <constructor-arg index="3" value="${zookeeper_watch_interval}"/>
</bean>
```

### 4.2 更改节点

```java
import org.apache.zookeeper.*;
import org.apache.zookeeper.data.Stat;

import java.io.IOException;

/**
 * Desc:
 * Author:sunguoli
 * Date:15/10/6
 * Time:下午2:04
 */
public class zkConfigServer implements Watcher{
    ZooKeeper zkServer = null;
    String zkNode;

    zkConfigServer(String zkServer, String zkNode, int zkTimeout) {
        this.zkNode = zkNode;
        try {
            this.zkServer = new ZooKeeper(zkServer, zkTimeout, this);
            Stat st = this.zkServer.exists(zkNode, true);
            if (st == null) {
                this.zkServer.create(zkNode, new byte[0], ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
            }
        } catch (IOException e) {
            e.printStackTrace();
            this.zkServer = null;
        } catch (KeeperException e) {
            e.printStackTrace();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    void updateConfig(String str) {
        try {
            Stat s = this.zkServer.exists(this.zkNode, true);
            this.zkServer.setData(this.zkNode, str.getBytes(), s.getVersion());
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    @Override public void process(WatchedEvent event) {
        System.out.println(event.toString());
        try {
            this.zkServer.exists(zkNode, true);
        } catch (KeeperException e) {
            e.printStackTrace();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    public static void main(String[] args) {
        zkConfigServer configServer = new zkConfigServer("localhost:2181", "/node", 2000);
        configServer.updateConfig("haha");
    }

}
```

可以运行代码可以看到/node节点的值被设置成了"haha",或者当节点不存在时，会首先创建节点。



参考文章：

官方文档，very good! http://zookeeper.apache.org/doc/r3.4.6/zookeeperStarted.html

[https://www.ibm.com/developerworks/cn/opensource/os-cn-zookeeper/](https://www.ibm.com/developerworks/cn/opensource/os-cn-zookeeper/)

[http://qindongliang.iteye.com/blog/1985087](http://qindongliang.iteye.com/blog/1985087)

[http://blog.javachen.com/2013/08/23/publish-proerties-using-zookeeper.html](http://blog.javachen.com/2013/08/23/publish-proerties-using-zookeeper.html)

[http://blog.csdn.net/copy202/article/details/8957939](http://blog.csdn.net/copy202/article/details/8957939)

[http://blog.fens.me/hadoop-zookeeper-intro/](http://blog.fens.me/hadoop-zookeeper-intro/)

http://shiyanjun.cn/archives/474.html