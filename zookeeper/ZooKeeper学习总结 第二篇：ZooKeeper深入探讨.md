# ZooKeeper学习总结 第二篇：ZooKeeper深入探讨
其实zookeeper系列的学习总结很早就写完了，这段时间在准备找工作的事情，就一直没有更新了。下边给大家送上，文中如有不恰当的地方，欢迎给予指证，不胜感谢！。
## 1. 数据模型
### 1.1. 只适合存储小数据
Zk维护着一个逻辑上的树形层次结构，树中的节点称为znode，个znode都有一个ACL（权限控制）。Zookeeper是被设计用来协调服务的，因此znode里存储的都是小数据，而不是大容量的数据，数据容量一般在1MB范围内。
### 1.2. 操作的原子性
Znode的数据读写是原子的，要么读或写了完整的数据，要么就失败，不会出现只读或写了部分数据。
### 1.3. Znode的路径
和Unix中的文件系统路径格式很想，但是只支持绝对路径，不支持相对路径，也不支持点号（”.”和”..”）。
### 1.4. 短暂的znode和持久的znode
Znode有两种类型：短暂的和持久的。短暂的znode生命周期仅限创建它的客户端与服务器端之间的连接没有断开，客户端断开连接后，znode将会被删除。
### 1.5. 顺序znode
名称中包含Zookeeper指定顺序号的znode。若在创建znode时设置了顺序标识，那么该znode被创建后，名字的后边将会附加一串数字，该数字是由一个单调递增的计数器来生成的。例如，创建节点时传入的path是”/aa/bb”，创建后的则可能是”/aa/bb0002”，再次创建后是”/aa/bb0003”。

Znode的创建模式CreateMode有四种，分别是：**EPHEMERAL**（短暂的znode）、**EPHEMERAL_SEQUENTIAL**（短暂的顺序znode）、**PERSISTENT**（持久的znode）和**PERSISTENT_SEQUENTIAL**（持久的顺序znode）。如果您已经看过了上篇博文，那么这里的api调用应该是很好理解的，见：http://www.cnblogs.com/leocook/p/zk_0.html。
### 1.6. 观察
这部分在上篇博文中已经做了详细的说明，包括连接的观察和znode的观察,这部分在构建一个稳定的zookeeper应用中有着很重要的作用，具体会在下边说到。
## 2.  ACL
即：Access Control List（访问控制列表）。Znode被创建时带有一个ACL列表，zk提供了下边**三种身份验证模式**：
**digest**
用户名+密码验证。
**host**
客户端主机名hostname验证。
**ip**
客户端的IP验证。
**auth**
使用sessionID验证
**world**
无验证，默认是无任何权限，该模式较为特殊，在给zk连接添加ACL中会说到

ACL权限对应如下表：
![091618429627580](.\img\091618429627580.png)
在设置ACL时，可以给zk客户端和服务器端的连接设置ACL，也可以在创建znode时，给znode设置ACL，在创建了znode后，如果有zk客户端来操作znode，只有满足权限要求时，才能完成相对应的操作：
### 2.1. 给ZK连接添加ACL
可使用zk对象的addAuthInfo()方法来添加验证模式，如使用digest模式进行身份验证：zk.addAuthInfo(“digest”,”username:passwd”.getBytes());
在zookeeper对象被创建时，初始化会被添加world验证模式。world身份验证模式的验证id是”anyone”。
若该连接创建了znode，那么他将会被添加auth身份验证模式的验证id是””，即空字符串，这里将使用sessionID进行验证。
### 2.2. 给znode设置ACL
**自己创建ACL**
创建ACL对象时，可用**ACL类的构造方法ACL(int perms, Id id)**：
其中参数perms表示权限，在接口org.apache.zookeeper.ZooDefs.Perms中有相关的常量：READ、WRITE、CREATE、DELETE、ALL和ADMIN，它们值如下表：
![091618429627580](.\img\091618429627580.png)

**Id参数是验证模式**，可用构造方法Id(String scheme, String id)来创建。参数scheme是验证模式，digest、host或ip，id是对应的验证，digest对应用户名和密码对，如“user:passwd”；host对应主机名，如”localhost”；ip对应ip地址，如”192.168.1.120”。

**使用api中预设的ACL**
在创建znode时可以设置该znode的ACL列表。接口org.apache.zookeeper.ZooDefs.Ids中有一些已经设置好的权限常量，例如：
(1)、**OPEN_ACL_UNSAFE**：完全开放。事实上这里是采用了world验证模式，由于每个zk连接都有world验证模式，所以znode在设置了OPEN_ACL_UNSAFE时，是对所有的连接开放。
(2)、**CREATOR_ALL_ACL**：给创建该znode连接所有权限。事实上这里是采用了auth验证模式，使用sessionID做验证。所以设置了CREATOR_ALL_ACL时，创建该znode的连接可以对该znode做任何修改。
(3)、**READ_ACL_UNSAFE**：所有的客户端都可读。事实上这里是采用了world验证模式，由于每个zk连接都有world验证模式，所以znode在设置了READ_ACL_UNSAFE时，所有的连接都可以读该znode。

## 3. 运行模式
Zookeeper有两种运行模式：独立模式（standalone mode）和复制模式（replicated mode）。
### 3.1. 独立模式
只有一个zookeeper服务实例，不可保证高可靠性和恢复性，可在测试环境中使用，生产环境不建议使用。
### 3.2. 复制模式
复制模式也就是集群模式，有多个zookeeper实例在运行，建议多个zk实例是在不同的服务器上。集群中不同zookeeper实例之间数据不停的同步。有半数以上的实例保持正常运行，zk服务就能正常运行，例如：有5个zk实例，挂了2个，还剩3个，依然可以正常工作；如有6个zk实例，挂了3个，则不能正常工作。
每个znode的修改都会被复制到超过半数的机器上，这样就会保证至少有一台机器会保存最新的状态，其余的副本最终都会跟新到这个状态。Zookeeper为实现这个功能，使用了Zab协议，该协议有两个可以无限重复的阶段：
**选举领导**
集群中所有的zk实例会选举出来一个“领导实例”（leader），其它实例称之为“随从实例”（follower）。如果leader出现故障，其余的实例会选出一台leader，并一起提供服务，若之前的leader恢复正常，便成为follower。选举follower是一个很快的过程，性能影响不明显。
Leader主要功能是协调所有实例实现写操作的原子性，即：所有的写操作都会转发给leader，然后leader会将更新广播给所有的follower，当半数以上的实例都完成写操作后，leader才会提交这个写操作，随后客户端会收到写操作执行成功的响应。
**原子广播**
上边已经说到：所有的写操作都会转发给leader，然后leader会将更新广播给所有的follower，当半数以上的实例都完成写操作后，leader才会提交这个写操作，随后客户端会收到写操作执行成功的响应。这么来的话，就实现了客户端的写操作的原子性，每个写操作要么成功要么失败。逻辑和数据库的两阶段提交协议很像。
### 3.3. 复制模式下的数据一致性
Znode的每次写操作都相当于数据库里的一次事务提交，每个写操作都有个全局唯一的ID，称为：zxid（ZooKeeper Transaction）。ZooKeeper会根据写操作的zxid大小来对操作进行排序，zxid小的操作会先执行。zk下边的这些特性保证了它的数据一致性：
**顺序一致性**
任意客户端的写操作都会按其发送的顺序被提交。如果一个客户端把某znode的值改为a，然后又把值改为b（后面没有其它任何修改），那么任何客户端在读到值为b之后都不会再读到a。
**原子性**
这一点再前面已经说了，写操作只有成功和失败两种状态，不存在只写了百分之多少这么一说。
**单一系统映像**
客户端只会连接host列表中状态最新的那些实例。如果正在连接到的实例挂了，客户端会尝试重新连接到集群中的其他实例，那么此时滞后于故障实例的其它实例都不会接收该连接请求，只有和故障实例版本相同或更新的实例才接收该连接请求。
**持久性**
写操作完成之后将会被持久化存储，不受服务器故障影响。
**及时性**
在对某个znode进行读操作时，应该先执行sync方法，使得读操作的连接所连的zk实例能与leader进行同步，从而能读到最新的类容。

**注意**：sync调用是异步的，无需等待调用的返回，zk服务器会保证所有后续的操作会在sync操作完成之后才执行，哪怕这些操作是在执行sync之前被提交的。

## 4. 提高ZooKeeper应用的容错
分布式环境是很复杂的，网络的不可靠、单点故障等问题都是经常发生的。那么在构建一个分布式应用程序时，这些问题都是需要慎重考虑的。因此，如何构建一个可复原的分布式应用将成为一个值得讨论的话题。Java api中每个异常都对应一类故障模式，下边我们将会以Java api中的异常为例来讨论ZooKeeper应用程序中可能会出现的一些故障。

### 4.1. Java API中的一些常见异常
**InterruptedException异常**
若客户端的某操作被中断，则会抛出InterruptedException异常。抛出该异常时，不一定是出现故障，只能表明某个zookeeper操作被中断而已。
**KeeperException异常**
服务器发出错误信号或是服务器存在通信故障。该类现在共有21个子类， 分为3大类：

不可恢复的异常发生时，所有的短暂znode都将会丢失，只有程序中显示的重建zk连接，并重建znode的状态。例如：会话过期会抛出异常SessionExpiredException，身份验证失败会抛出异常AuthFailedException。

(1)、状态异常
当一个客户端对zk的某操作失败时，就会出现状态异常。例如：更新数据时所指定的版本号不正确就会抛出异常BadVersionException、若在短暂的znode下创建子节点则会抛出异常NoChildrenForEphemeralsException。

(2)、可恢复的异常
那些在zk会话中可以恢复的异常叫可恢复的异常。当丢失zk连接时就会抛出异常ConnectionLossException，这时zk会自动尝试重新连接，以确保会话的完整性。Zk无法判断ConnectionLossException异常相关的操作是否成功执行，有可能出现只完成部分，那么是否重新执行刚才的操作就得知道该操作是否是等幂的。
等幂操作是指一次或多次执行都会产生相同结果的操作；非等幂操作是指一次或多次执行会产生不相同结果的操作。非等幂操作就不能盲目操作了。
写操作里有创建、删除、修改。在一个分布式环境中，删除zk里的znode或是修改znode的数据是等幂的，只有创建znode可能不是等幂的，创建顺序znode就是一个非等幂的操作。
那么怎么样才能避免创建顺序znode不会出现重复创建呢？下面我来展开讨论：

**假设的场景：**
**客户端**：客户端任务是在连接到zk服务端时会只创建一个顺序znode；
**ConnectionLossException**：抛出ConnectionLossException异常重新连接后会话没有失效， 但是zk无法判断创建znode的操作是否成功。
我们知道顺序znode的节点名称格式是形如”znodeName<sequentialNumber>”,zk客户端和服务器端的会话有个全局唯一的sessionID，我们可以把sessionID加入znode的名称中，形如：” znodeName<sessionID><sequentialNumber>”，sequentialNumber是相对于父znode唯一的。这样我们在创建某个znode之前先判断一下父znode下有无名称形如” znodeName<sessionID>“这样字符开头的子znode，就能确保每个客户端连接只创建一个znode。
这种场景会在什么时候会遇到呢？在我们要实现一个分布式锁的时候，核心思想之一就在这里。那么问题来了，什么是分布式锁呢？后面会有独立的博文来讲解关于它的代码实现。

(3)、不可恢复的异常
不可恢复的异常发生时，所有的短暂znode都将会丢失，只有程序中显示的重建zk连接，并重建znode的状态。例如：会话过期会抛出异常SessionExpiredException，身份验证失败会抛出异常AuthFailedException。

(4)、异常捕捉处理
每个子类对应一种异常状态，且每个子类都对应一个关于错误类型的信息代码，可以通过code方法拿到。 处理该种异常有两种办法：
1、通过检测错误代码（可调用code方法老获取）来确定是哪种异常，再决定应该采取何种补救措施；
2、通过追捕等价的KeeperException异常，然后再每段捕捉代码中执行相应的操作。

### 4.2. 构建可靠的zookeeper应用
上面说到了zk服务器端可能出现一些网络故障或单点故障登，那么怎么编写一个可靠的zk客户端程序来应对可能不稳定的zk实例呢？这里我们向一个znode写数据为例，来实现它：
```JAVA
/**
 * 显示配置
 * @throws KeeperException 服务器发出错误信号或是服务器存在通信故障。该类现在共有21个子类，
 * 分为3大类：<br/>
 * 1、状态异常(如：BadVersionException、NoChildrenForEphemeralsException)；
 * 2、可恢复的异常（如：ConnectionLossException）；
 * 3、不可恢复的异常（如：SessionExpiredException、AuthFailedException）。
 * 每个子类对应一种异常状态，且每个子类都对应一个关于错误类型的信息代码，可以通过code方法拿到。
 * 处理该种异常有两种办法：<br/>
 * 1、通过<b>检测错误代码</b>来决定应该采取何种补救措施；<br/>
 * 2、通过<b>追捕等价的KeeperException异常</b>，然后再每段捕捉代码中执行相应的操作。
 * @throws InterruptedException zookeeper操作被中断。<b>并不一定就是出现故障，只能表明相对应的操作被取消</b>。
 */
public static void write(String path, String value) throws KeeperException, InterruptedException {
    int retries = 0;
    
    while (true) {
        try {
            Stat stat = zk.exists(path, false);
            if(stat == null){
                zk.create(path, value.getBytes(CHARSET), Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
            }else {
                zk.setData(path, value.getBytes(CHARSET), -1);
            }
            break;
        } catch(KeeperException.SessionExpiredException e){
            //TODO 此处会话过期，抛出异常，由上层调用来重新创建zookeeper对象
            throw e;
        }catch(KeeperException.AuthFailedException e){
            //TODO 此处身份验证时，抛出异常，由上层来终止程序运行
            throw e;
        }catch (KeeperException e) {
            //检查有没有超出尝试的次数
            if(retries == MAXRETRIES){
                throw e;
            }
            retries++;
            TimeUnit.SECONDS.sleep(RETRY_PERIOD_SECONDS);
        }
    }
}
```
如果您是一名Java开发人员，那么我觉得上面的这些代码没什么好解释的了。下边看上层调用是怎么处理的：

```JAVA
int flag = 0;

while (true) {
    try {
        write(path, value);
        break;
    } catch (KeeperException.SessionExpiredException e) {
        // TODO: 重新创建、开始一个新的会话
        e.printStackTrace();
        zk = new ZooKeeper(hosts, SESSION_TIMEOUT, this);
    } catch (KeeperException e) {
        // TODO 尝试了多次，还是出错，只有退出了
        e.printStackTrace();
        flag = 1;
        break;
    }catch(KeeperException.AuthFailedException e){
        //TODO 此处身份验证时，终止程序运行
        e.printStackTrace();
        flag = 1;
        break;
    } catch (IOException e) {
        // TODO 创建zookeeper对象失败，无法连接到zk集群
        e.printStackTrace();
        flag = 1;
        break;
    } 
}

System.exit(flag);
```
关于编写一个可恢复的zookeeper应用，这一块理解了，其它地方应该就是触类旁通了。
后边的博客将会更新几个zookeeper开发实例，例如分布式配置系统、分布式锁的实现。