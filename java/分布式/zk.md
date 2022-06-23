# Zookeeper

## 基础理论

### paxos算法和ZAB理论
	在不可靠的信道上通过消息传递的方式达成一致性时不可行的（不存在拜占庭将军问题：即士兵传递的消息有可能是不可靠的）
#### 三种角色
- proposer 提案者
- acceptor 表决者
- learner 同步者
同一个节点可能同时充当多个角色
#### 一致性体现
- 每个提案者在提出提案时，首先会获取一个全局唯一的、递增的提案编号N，即在整个集群中唯一的编号N
- 每个表决者在accept某个提案后，会将该提案的编号N记录在本地，这样表决者本地就存在一个已经accept的最大提案编号maxN，每个表决者仅accept编号大于本地maxN的提案
- 在众多提案中，仅有一个会被选定
- 提案被选定后，其他服务器会同步这个提案到本地
- 没有提案被提出，就不会有提案被选中

#### 过程描述
提案者发出提案，表决者接收到了以后，如果accept就返回一个ACK，半数通过后，提案生效，同步者同步提案到本地

##### prepare阶段
- 提案者准备提交一个编号为N的提议，于是向其他所有表决者发送prepare(N)请求，用于试探集群是否支持这个提议
- 每个表决都保存着自己曾经accept的提议中最大的编号maxN，当一个表决者接收到其他主机发送过来的prepare(N)请求时，会比较N与maxN的值，
	- 如果N小于maxN，说明这个提案已经过时，采取不回应或者回应error来拒绝提案
	- 如果N大于maxN，说明提案可以接受，表决者会记录N，并将已经accept过的最大编号提案Proposal(myid, maxN, value)反馈给提案者，以向提案者展示自己支持的提案信息，第一个参数表示表决者的标识id，maxN表示接受过的最大提案编号maxN，第三个参数表示提案内容，如果当前表决者还没有accept过任何提案，就会返回Proposal(myid, null, null) 给提案者
- 在prepare阶段N不可能等于maxN，这是由N的生成机制决定的，N是全局递增的。
##### accept阶段
- 当提案者发出prepare(N)后，收到超过半数的accepter反馈，那么提案者就把真正的提案proposal(myid, N, value)发送给所有的表决者
- 当表决者收到提案后，会再次将层accept过的maxN（或者曾经记录下的prepare的最大编号）与收到的N进行比较，如果N大于这些编号，则当前表决者accept该提案，并反馈给提案者，若果小于，则返回拒绝这个提案
- 若提案者没有收到半数以上的表决者accept，则有两种可能的结果产生，一种是放弃该提案不再提出，二是重新进入prepare阶段，递增提案号，重新提出prepare请求
- 若提案者收到的反馈数量超过了半数，则会向外广播两类消息：（commit信号）
	- 向曾经accept其提案的表决者发送“可执行同部数据信号“，即让他们执行其层接收到的提案
	- 向未曾向其发送accept反馈的表决者发送“提案+可执行数据同步信号”，即让他们接受提案后马上执行

##### 问题
1、为什么第二个阶段还要进行比较N
	在一个节点的（prepare）提案获得了多数节点的确认后还没有accpet，其他的节点也获得了多数节点的确认（prepare）；这时候就有两个提案获得了通过，所以在prepare阶段还需要比较提案编号
2、为什么发送commit信号量的时候不需要比较
	commit阶段表决者只比较提案编号，不更新本地最大提案编号

###### 活锁问题
由于某些条件没有被满足，处于尝试-失败-尝试-失败的循环之中，活锁有可能自行解开

### ZAB协议
zk原子消息广播协议，支持崩溃恢复的广播协议
为了避免单点问题，zk是以集群形式存在的；
即：zk使用单一主进程来接受处理客户端的事务请求，就是写请求；数据变更以后，使用ZAB原子广播，以提案广播的形式通知从节点 
#### 三类角色
- leader 事务请求的唯一处理者，也可以处理读请求
- follower 可以直接处理客户端请求，不会处理事务请求，只会将事务请求转发给leader；对leader发起的事务提案具有表决权；同步leader的事务处理结果；可以投票，也可以被选举
- observer：不参与选举过程的follower，用于缓解集群的读压力；数量一般与follower数量差不多，过多的observer节点会增大leader的同步压力和网略压力；但不会增加事务操作的压力，**observer同步数据的时间小于follower，follower同步结束后，observer的同步也将结束；这样leader中就存在两个observer列表，一个是同步过数据的列表（service），一个是全部observer列表（all）**；service列表是动态的，observer会通过心跳与leader进行连接，连接成功就进行数据同步没同步完成后返回ACK，leader就会将这个节点添加到service列表；读操作比较多的系统可以适量增加observer节点
- QuorumServer：Particiant 法定主机，参与者，具有表决权的服务器
- learner：从leader中同步数据的server；=follower+observer
- quorumServer：法定服务器：leader+follower
#### 三个重要数据
- zxid：long类型数据，高32位表示epoch，低32位表示xid
- epoch（时期、年号）：每一个leader选举结束后都会生成一个epoch，并会通知到集群中所有其他的server，包含follower和observer
- xid：事务id，是一个流水号
	
#### 三种模式
ZAB协议中对zkServer状态描述的三种模式，他们之间没有明显的界限，相互交织在一起
- 恢复模式：集群启动过程中，或者leader崩溃后，系统都需要进入恢复模式，以恢复系统对外的服务能力，包含leader选举和初始化同步
- 广播模式：分为两类：初始化广播和更新广播
- 同步模式：分为初始化同步和更新同步

#### 四种状态
各节点的正常工作状态
- LOOKING
- FOLLOWER
- OBSERVING
- LEADING
##### 初始化广播
完成leader选举后，此时的leader还是一个准leader，还要经过初始化广播才能真正成为leader；
1、leader会为每一个learner创建一个FIFO队列；保证向learner发送提案的有序
2、将未被同步的事务封装为Proposal；
3、发送Proposal+COMMIT给learner
4、learner更新Proposal到本地
5、learner返回ACK给leader
6、将收到反馈的Follower添加到一个Follower队列


##### 更新广播
1、leader将事务封装为Proposal
2、向follower发送
3、follower判断proposal的zxid是否大于本地的max-zxid，如果大于，将Proposal保存到本地事务日志
4、follower向leader返回ACK
5、判断是否收到过半的ACK，如果过半，向learner发送消息
6、向所有的follower发送COMMIT；向所有Observer发送Proposal
若leader收到follower反馈没有过半，就认为同步失败，进行重新同步


##### 集群中的learner接收到事务请求 更新广播详细描述
	会将事务请求转发给leader服务器，然后执行以下过程
1、leader接收到事务请求，为事务赋予一个全局64位自增的id，zxid；通过zxid对事务的有序性进行管理；然后将事务封装为一个proposal。
2、leader根据follower列表获取所有的follower（队列），然后将proposal发送给队列中的各个follower；
3、follower收到提案后，将提案zxid和本地事务日志中的最大的zxid进行比较，如果提案的zxid比较大，将当前提案记录到本地事务日志中，并向leader返回ACK
4、leader收到过半的ACK后，就向所有的follower发送commit消息，向所有observer发送proposal
5、follower收到commit消息，就会将日志中的事务正式更新到本地，当observer收到proposal后，直接将事务更新到本地
6、无论是Follower还是observer，同步完成后都要向leader发送成功的ACK


#### 四种状态
zk集群中每一台主机，在不同阶段会处于不同状态，每一台主机具有四种状态
- looking：刚启动zookeeper时，正在进行选举leader的过程的server所处状态
- following： 成为follower后的server所处状态
- observing： 在一个很大的集群中，没有投票权的server所处状态
- leading：已被选举为leader后的server所处状态

### 恢复模式需要遵守的原则
#### leader主动出让原则
leader收到follower心跳的数量没有过半，就认为自己与集群连接出现问题，进入looking状态，去查找leader，防止出现脑裂（两个leader）
其他的server中有一半心跳失败认为丢失了leader，会发出新的leader选举

#### 已被处理的消息不能丢弃；被丢弃的消息不能再现
	1、`leader`收到超过半数的`follower`的`ACK`，会向`follower`广播`commit`消息；节点收到`connit`会在本地执行写操作，并向客户端返回；
	如果部分`follower`收到`commit`之前，`leader`就挂了；这导致部分节点执行了该事务，而其他节点没有执行；当新的`leader`被选举出来后，需要保证所有的节点都执行过这个事务
	所以只有`zxid`最大的节点会当选`leader`
	2、所有的`follower`都没有收到`commit`，`leader`挂了，这时候选举出来的新`leader`，也不能重新提交这个事务，这个事务需要丢弃

#### 可能存在的脑裂情况
多机房部署的情况下，若出现了网络连接问题，形成多个分区，可能会出现脑裂，导致数据不一致

zk的过半机制已经很好的防止了脑裂的情况出现

	Zookeeper通过内部心跳机制来确定leader的状态，一旦leader出现意外Zookeeper能很快获悉并且通知其他的follower，其他flower在之后作出相关反应，这样就完成了一个切换，这种模式也是比较通用的模式，基本大部分都是这样实现的。但是这里面有个很严重的问题，如果注意不到会导致短暂的时间内系统出现脑裂，因为心跳出现超时可能是leader挂了，但是也可能是zookeeper节点之间网络出现了问题，导致leader假死的情况，leader其实并未死掉，但是与ZooKeeper之间的网络出现问题导致Zookeeper认为其挂掉了然后通知其他节点进行切换，这样follower中就有一个成为了leader，但是原本的leader并未死掉，这时候client也获得leader切换的消息，但是仍然会有一些延时，zookeeper需要通讯需要一个一个通知，这时候整个系统就很混乱可能有一部分client已经通知到了连接到新的leader上去了，有的client仍然连接在老的leader上，如果同时有两个client需要对leader的同一个数据更新，并且刚好这两个client此刻分别连接在新老的leader上，就会出现很严重问题。
- 假死：由于心跳超时（网络原因导致的）认为leader死了，但其实leader还存活着。
- 脑裂：由于假死会发起新的leader选举，选举出一个新的leader，但旧的leader网络又通了，导致出现了两个leader ，有的客户端连接到老的leader，而有的客户端则连接到新的leader。


#### zk是如何解决脑裂的

要解决Split-Brain脑裂的问题，一般有下面几种种方法：
Quorums (法定人数) 方式: 比如3个节点的集群，Quorums = 2, 也就是说集群可以容忍1个节点失效，这时候还能选举出1个lead，集群还可用。比如4个节点的集群，它的Quorums = 3，Quorums要超过3，相当于集群的容忍度还是1，如果2个节点失效，那么整个集群还是无效的。这是zookeeper防止"脑裂"默认采用的方法。
采用Redundant communications (冗余通信)方式：集群中采用多种通信方式，防止一种通信方式失效导致集群中的节点无法通信。
Fencing (共享资源) 方式：比如能看到共享资源就表示在集群中，能够获得共享资源的锁的就是Leader，看不到共享资源的，就不在集群中。
仲裁机制方式。
启动磁盘锁定方式。

要想避免zookeeper"脑裂"情况其实也很简单，在follower节点切换的时候不在检查到老的leader节点出现问题后马上切换，而是在休眠一段足够的时间，确保老的leader已经获知变更并且做了相关的shutdown清理工作了然后再注册成为master就能避免这类问题了，这个休眠时间一般定义为与zookeeper定义的超时时间就够了，但是这段时间内系统可能是不可用的，但是相对于数据不一致的后果来说还是值得的。

1、zooKeeper默认采用了Quorums这种方式来防止"脑裂"现象。即只有集群中超过半数节点投票才能选举出Leader。这样的方式可以确保leader的唯一性,要么选出唯一的一个leader,要么选举失败。在zookeeper中Quorums作用如下：
1]  集群中最少的节点数用来选举leader保证集群可用。
2] 通知客户端数据已经安全保存前集群中最少数量的节点数已经保存了该数据。一旦这些节点保存了该数据，客户端将被通知已经安全保存了，可以继续其他任务。而集群中剩余的节点将会最终也保存了该数据。

假设某个leader假死，其余的followers选举出了一个新的leader。这时，旧的leader复活并且仍然认为自己是leader，这个时候它向其他followers发出写请求也是会被拒绝的。因为每当新leader产生时，会生成一个epoch标号(标识当前属于那个leader的统治时期)，这个epoch是递增的，followers如果确认了新的leader存在，知道其epoch，就会拒绝epoch小于现任leader epoch的所有请求。那有没有follower不知道新的leader存在呢，有可能，但肯定不是大多数，否则新leader无法产生。Zookeeper的写也遵循quorum机制，因此，得不到大多数支持的写是无效的，旧leader即使各种认为自己是leader，依然没有什么作用。

zookeeper除了可以采用上面默认的Quorums方式来避免出现"脑裂"，还可以可采用下面的预防措施：
2、添加冗余的心跳线，例如双线条线，尽量减少“裂脑”发生机会。
3、启用磁盘锁。正在服务一方锁住共享磁盘，"裂脑"发生时，让对方完全"抢不走"共享磁盘资源。但使用锁磁盘也会有一个不小的问题，如果占用共享盘的一方不主动"解锁"，另一方就永远得不到共享磁盘。现实中假如服务节点突然死机或崩溃，就不可能执行解锁命令。后备节点也就接管不了共享资源和应用服务。于是有人在HA中设计了"智能"锁。即正在服务的一方只在发现心跳线全部断开（察觉不到对端）时才启用磁盘锁。平时就不上锁了。
4、设置仲裁机制。例如设置参考IP（如网关IP），当心跳线完全断开时，2个节点都各自ping一下 参考IP，不通则表明断点就出在本端，不仅"心跳"、还兼对外"服务"的本端网络链路断了，即使启动（或继续）应用服务也没有用了，那就主动放弃竞争，让能够ping通参考IP的一端去起服务。更保险一些，ping不通参考IP的一方干脆就自我重启，以彻底释放有可能还占用着的那些共享资源

### leader选举
- myid：即serverid，是zk集群中唯一表示
- 逻辑时钟：logicalclock；选举时被称为logicalclock；选举结束后成为epoch；是同一个值，在不同阶段的不同名称

#### 选举算法

-------集群启动之后的选举



### 集群容灾
5个节点的集群，最多允许宕机两台，6台也是一样，所以在容灾能力上是一样的，但是6个节点的系统吞吐量是比5台要好的；
#### 三机房部署
在三个机房部署集群，每个集群的节点数量都要少于总量的一半，这样，任意一个机房出现意外，其他两个机房都有过半的节点，可以形成决议
#### 双机房部署



### CAP
- 一致性Consistency：分布式系统中，多个节点保持数据一致；
- 可用性availability：在有限时间内作出响应
- partition tolerance：分区容错性

分布式系统只能保证其中两项，CP或者AP

#### BASE理论
BASE=Basically available基本 可用、soft state 软状态、Eventually consistent最终一致性
- 基本可用：分布式系统出现不可预知的故障时，允许损失部分可用性
- 软状态：允许系统数据存在中间状态，并认为该中间状态的存在影响系统的整体可用性，即：允许存在一定的延时
- 最终一致性：在经过一段时间的同步后，最终能够达到数据一致

