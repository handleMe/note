##  设置过期时间
### 设置过期时间
`setex 命令`
`通过expire或者pexpire命令；前者以秒为单位，后者以毫秒为单位；expireat和pexpireat以timestamp来设置过期秒数或者毫秒数。实际上这些命令都是通过pexpireat来实现`
### 查询过期时间
`ttl key` 查看还有多少秒过期
`pttl key`查看还有多少毫秒过期

### 移除过期时间
`persist key` 移除后ttl返回-1

### 删除策略
- 定时删除，通过使用定时器，在键过期时尽快删除，CPU不友好
- 惰性删除，程序只在取出键的时候进行过期检查并删除键；内存不友好
- 定期删除：每过一段时间进行一次过期键删除的操作

## 备份策略
两种备份文件共存时，优先使用AOF进行还原；因为AOF备份的更新频率更高，丢失数据更少
### RDB
保存当前的数据
stop-writes-on-bgsave-error yes：当bgsave出现错误时，Redis是否停止执行写命令；设置为yes，则当硬盘出现问题时，可以及时发现，避免数据的大量丢失；设置为no，则Redis无视bgsave的错误继续执行写命令，当对Redis服务器的系统(尤其是硬盘)使用了监控时，该选项考虑设置为no
rdbcompression yes：是否开启RDB文件压缩
rdbchecksum yes：是否开启RDB文件的校验，在写入文件和读取文件时都起作用；关闭checksum在写入文件和启动文件时大约能带来10%的性能提升，但是数据损坏时无法发现
dbfilename dump.rdb：RDB文件名
dir ./：RDB文件和AOF文件所在目录
#### 手动触发
通过save命令或者bgsave命令来触发；save会阻塞Redis的进程，直到RDB备份完成
bgsave会fork一个进程进行持久化，不会阻塞主进程

#### 自动备份
通过配置save参数来完成
save 100 1表示100秒内进行了至少一次修改就进行备份

#### 其他触发
从节点进行全量复制，会触发主节点的bgsave，并将rdb文件发给从节点
### AOF
以命令的形式进行备份
appendonly yes  ---- 打开aof设置，同一时候将快照功能置于低优先级的位置
appendfilename "appendonly.aof"文件名称
● no-appendfsync-on-rewrite no：在重写 AOF 文件的过程中，是否禁止 fsync。如果这个参数值设置为 yes（开启），则可以减轻重写 AOF 文件时 CPU 和硬盘的负载，但同时可能会丢失重写 AOF 文件过程中的数据；需要在负载与安全性之间进行平衡。
● auto-load-truncated yes：当 AOF 文件结尾遭到损坏时，Redis 在启动时是否仍加载 AOF 文件。
#### AOF缓存刷入磁盘的设置
appendfsync no
        当设置appendfsync为no的时候，Redis不会主动调用fsync去将AOF日志内容同步到磁盘。所以这一切就全然依赖于操作系统的调试了。对大多数Linux操作系统。是每30秒进行一次fsync，将缓冲区中的数据写到磁盘上。
appendfsync everysec
        当设置appendfsync为everysec的时候。Redis会默认每隔一秒进行一次fsync调用，将缓冲区中的数据写到磁盘。可是当这一次的fsync调用时长超过1秒时。Redis会採取延迟fsync的策略。再等一秒钟。也就是在两秒后再进行fsync，这一次的fsync就无论会运行多 长时间都会进行。这时候因为在fsync时文件描写叙述符会被堵塞，所以当前的写操作就会堵塞。在绝大多数情况下。Redis会每隔一秒进行一 次fsync。
	在最坏的情况下，两秒钟会进行一次fsync操作。这一操作在大多数数据库系统中被称为group commit。就是组合多次写操作的数据，一次性将日志写到磁盘。

appendfsync always
        置appendfsync为always时，每一次写操作都会调用一次fsync，这时数据是最安全的，当然。因为每次都会运行fsync。所以其性能也会受到影响。

#### AOF重写
触发：根据以下连个参数判断
● auto-aof-rewrite-percentage 100：指定 Redis 重写 AOF 文件的条件，默认为 100，它会对比上次生成的 AOF 文件大小。如果当前 AOF 文件的增长量大于上次 AOF 文件的 100%，就会触发重写操作；如果将该选项设置为 0，则不会触发重写操作。
● auto-aof-rewrite-min-size 64mb：指定触发重写操作的 AOF 文件的大小，默认为 64MB。如果当前 AOF 文件的大小低于该值，此时就算当前文件的增量比例达到了 auto-aof-rewrite-percentage 选项所设置的条件，也不会触发重写操作。换句话说，只有同时满足以上这两个选项所设置的条件，才会触发重写操作。

为了保证重写的时候数据一致，主进程需要：
- 执行客户端命令
- 执行后的命令写入到AOF缓冲区
- 执行后的命令追加到AOF重写缓冲区，等待刷入磁盘

### v4.0+的混合持久化

## 主从复制
slave of命令将当前服务器转变为指定服务器的从属服务器(slave server)
SLAVEOF 127.0.0.1 6379 
SLAVEOF NO ONE  将使得这个从属服务器关闭复制功能，并从从属服务器转变回主服务器，原来同步所得的数据集不会被丢弃

### salve过程：
- 从库向主库发送sync命令
- 主库接收到以后，执行bgsave命令记性rdb持久化，同时使用缓冲区将执行的命令写入
- 主库发送rdb文件给从库，从库载入rdb文件
- 主库将缓冲区的所有命令发送给从库；从库执行这些命令

### 命令传播
主服务器将改变的命令发送给从库执行

### 断线重连
v2.8之前
从库断线重连后，会给主库发送sync命令
v2.8之后
使用psync代替sync
psync有两种模式
- 完全同步，与sync一致
- 部分同步：同步从主库断开期间的命令（如果可以的话）
	- 主库与分库分别维护一个复制偏移量，通过比较偏移量来判断数据是否一致
	- 复制积压缓存区：FIFO队列，主库进行命令传播的同时也会将命令写入这个队列，默认大小1MB
	- 每个库都有自己的服务器ID
	- 从库重连后，psync发送偏移量给主库，偏移量之后的数据还在缓冲队列，就进行部分同步；不在了就进行全量同步

## Sentinel哨兵
	由一个或者多个sentinel组成的系统监控任意多个库及下属的从库
	主从服务器中，主库挂了，哨兵会通知从库终止复制，主库下线市场超过了用户设定，进行故障转移
	- 挑选一个从库升级为新的主库
	- 向其他的从库发送新的复制命令，成为新主库的从库
	- 当之前的主库上线，哨兵会将它设置为新主库的从库

### 启动哨兵
```
redis-sentinel .../sentinel.conf
或者redis-sentinel ..../sentinel.conf --sentinel
两者功能相同
```
- 初始化服务器
- 将普通服务器代码替换为sentinel专用代码
- 初始化sentinel状态
- 根据配置文件，初始化sentinel的坚实主服务器列表
- 创建连向主服务器的网络连接

### master下线和选举
sentinel每10秒发送INFO命令给主库，获取run_id和role（服务器角色），以及从库的信息
选举主库
- 将已下线主库的从库存到一个列表中
- 删除处于下线或者断线的库
- 删除列表中5S内没有恢复过主哨兵INFO命令的库
- 删除所有与已下线主库连接断开超过`down-after-millisenconds*10ms`的服务器
- 选取最大优先级的从库，若有多个优先级相同的库，选取偏移量最大的从库（数据最新），若有多个，选择ID最小的那个
- 向选举中从库发送slaveof no one命令，之后不断向这个库发送INFO，看到角色转变为master时，就知道选举成功了
- 让其他从库slaveof新的主库
- 下线的库上线后，设置为新主库的从库

### sentinel的leader选举
- 所有sentinel都有被选举为leader的资格
- 每次选举后，无论是否成功，configuration epoch都会+1，成为配置纪元，实际上是一个计数器
- 在一个纪元中，leader一旦设定，就不会修改
- 每个发现主哨兵下线的哨兵都会要求成为新的主哨兵
- 先到先得：最先向某个sentinel发送命令要求成为leader的sentinel，会被目标sentinel设置为主哨兵，之后同一个纪元的其他哨兵设置请求会被拒绝
- 某个sentinel被半数以上的节点设置为主哨兵，这个sentinel成为主哨兵
- 给定时间内，没有leader被选出，就会重复进行选举过程，直至选出


## 集群

- 节点：node通过`cluster meet <ip> <port>`连接各个节点，形成集群
- 启动节点：redis服务器启动的时候，根据cluster-enabled是否为yes来决定是否启用集群模式
- 在集群中也可以设置主从关系，主节点处理槽，从节点辅助主节点，并在主节点下线后代替其成为主节点继续处理命令

### 集群数据处理
    集群通过分片的方式来保存数据，集群中整个库分为16384个槽（slot）；
    每个键根据hash被分配到对应的槽；首先根据键值对的 key，按照CRC16 算法计算一个 16 bit 的值；然后，再用这个 16bit 值对 16384 取模，得到 0~16383 范围内的模数，每个模数代表一个相应编号的哈希槽。
    节点不仅记录自己处理那些槽，也将信息发送给其他节点；所以一个节点直到所有槽该由谁处理
    



#### 正常命令的处理
- 键所在的槽正好在当前节点，直接处理
- 键所在槽不再当前节点，向客户端返回MOVED（包含了槽所在服务器的信息）错误，引导客户端访问所在节点；客户端发送命令

#### 重新分片
重新划分槽的分布
	如果集群中有 N 个实例，那么，每个实例上的槽个数为 16384/N 个。当然， 我们也可以使用 cluster meet 命令手动建立实例间的连接，形成集群，再使用 cluster addslots 命令，指定每个实例上的哈希槽个数。
	在手动分配哈希槽时，需要把 16384 个槽都分配完，否则 Redis 集群无法正常工作
常见的有两种情况导致重新分片
- 在鸡群中有新增或者删除节点
- 为了负载均衡，redis需要把哈希槽在所有实例上重新分布一遍

#### ASK
ASK是重新分片的临时措施
ASK命令表示两层含义：第一，表明 Slot 数据还在迁移中；第二，ASK 命令把客户端所请求数据的最新实例地址返回给客户端，此时，客户端需要给新实例发送 ASKING命令，然后再发送操作命令。ASKING命令让这个实例允许执行客户端接下来发送的命令

#### 集群选举
	当一个从节点发现自己正在复制的主节点下线了
	- 该节点的从节点中会有一个被选中
	- 执行slaveof no one成为主节点
	- 将已下线主节点的槽指派撤销，并指派给自己
	- 向集群广播PONG消息，通知其他节点接管了主节点
	- 开始接受命令处理

**选举过程**：类似sentinel的leader选举，基于raft算法
- 某个节点开始故障转移，配置纪元自增
- 对于某个配置纪元，集群中每一个主节点都有一次投票权，第一个向主节点要求投票的从节点获得投票
- 从节点发现复制的主节点下线，向集群广播，要求主节点投票
- 一个从节点收到超过半数的支持票，成为新的主节点
- 没有选出主节点，就继续上述行为，直至选出为止
**心跳检测**
每个节点默认每秒从一致节点列表中随机5个节点发送PING消息，检测在线状态，此外若节点A最后一次收到B的PING消息，距离超过了A的cluster-node-timeout设置时长的一半，A会向B发送PING消息，防止长时间接收不到消息导致信息更新滞后

## 消息
- MEET消息：客户端接收到CLUSTER MEET时，向目标发送MEET，请求加入群组
- PING：心跳检测
- PONG：答应PING，也可以广播自己的PONG消息来立即刷新关于这个节点的认识
- FALL：主节点A判断主节点B进入FALL状态，A就会广播B的FALL消息，之后所有节点会将B标记为已下线
- PUBLISH：一个节点接收到，执行并向其他节点广播，接收到的节点都进行相应的操作

## 发布与订阅：
通过SUBSCRIBE命令，客户端可以订阅一个或者多个频道
subscribe "news.sport" "new.cctv"
当有其他客户端发布消息的时候，所有订阅者都会收到消息
publish "topic" "消息"

unsubscribe 可以退订一个或者多个频道

查看所有频道
`pubsub channels [pattern]`返回所有的频道

返回频道的订阅数
`pubsub numsub [channel]`

## 事务
mulit：标志着事务开始
	在事务状态期间，服务器不立即执行这个客户端的命令，将命令入队后，向客户端返回queued（不包含exec、discard、watch、mulit）
exec：执行事务，并返回所有结果
watch：监视任意数量的键；在exec执行的时候，检查被监视的键是否有变化，如果有直接失败

## LUA脚本

## 排序
redis的sort命令可以对列表、集合、有序集合进行排序
sort key 对key中元素进行排序并返回
ALPHA选项：sort key alpha 对一个包含字符串值得键排序，默认排序对象为数字
sort和by选项：
```
 > ZADD keyx 3.0 jack 3.5 peter 4.0 tom 添加元素及权重
 > 3
 > ZRANGE keyx 0 -1 按分值排序
 > MSET perter_number 1 tom_number 2 jack_number 3
以序号为权重进行排序
 > sort keyx by *_number
ASC DESC 选项
> sort key desc
limit选项 返回一部分元素
> limit <offset> <count>
> sort key alpha limit 0 4
get选项

store选项；排序默认不保存结果，可以使用store将结果保存在两一个键中
> sort stu alpha store soret_stu
> 
选线执行顺序
1.SORT <key> alpha desc by <...>
2.limit
3.get
4.store
选项的摆放顺序不会对结果由影响

```
## 二进制数组
- setbit用于指定偏移量上二进制位的值；`setbit key 0 1`；偏移量从0开始，值可以是0或者1
- getbit key offset 获取偏移量位置上的值
- bitcount key 统计数组中值为1的数量
	- 循环遍历
	- 查表 枚举每8位的所有情况，快速获取每8位
	- 计算汉明重量
- bitop 对多个数组进行与或非或者取反运算

```
x=[1,1,0,1]
y=[0,1,1,0]
z=[1,0,1,0]
> bitop and and-res x y z
> 1
> bitop or or-res x y z
> 1
> bitop xor-res x y z
> 1
> bitop not not-res x

```
## 慢查询日志
`slowlog-log-slower-than`选项指定执行时间超过多少毫秒的查询被记录在日志中
`slowlog-max-len`最多保存条数
```
> slowlog get 查看慢查询日志
> (integer) 4 日志唯一表示
  (integer)13242执行时的unix时间戳
  (integer)111执行时长 微秒
  "set"命令及参数
  "key"
  "value"
```
## 监视器
monitor
执行后，客户端会变成一个监视器，实时接收并打印出服务器处理请求的相关信息
