# redis
<a>https://blog.csdn.net/u011863024/article/details/107476187</a>
CAP

BASE：是为了解决关系数据库强一致性引起的问你而引起的可用性降低而提出的解决方案
- 基本可用：Basically Available
- 软状态：Soft state：
- 最终一致：Eventually consistent
通过让系统放松某一时候数据一致性的要求来换取系统整体整体伸缩性和性能

## 数据类型
- string
	- set
	- get
	- mset
	- mget
	- incr
		- incr key设置一个自增的数值，从1开始
	- incrby
	- decr
	- decrby
	- strlen：获取字符串长度
	- setnx key value 分布式锁
	- set key value [EX sen] [PX mill] [NX|XX]
		- EX：key在多少秒之后过期
		- PX：key在多少毫秒之后过期
		- NX：当key不存在时，才创建key，效果等同于setnx
		- XX：当key存在的时候，覆盖key
- list 列表，允许重复
	- LPUSH key value [value]...向列表左边添加元素
	- RPUSH key value [value]...向列表右边添加元素
	- LRANGE key start stop 查看列表
	- LLEN key 获取列表中元素的个数
- hash 相当于：Map<String, Map<Object, Object>>
	- HSET key field value一次设置一个字段值
	- HGET key field
	- HMSET key field value [field value...]
	- HMGET key field [field...]
	- hgetall key获取所有内容
	- hlen key获取大小
	- hdel key删除一个key
- set 不允许重复
	- sadd key member [member...]添加元素
	- srem key member [member...]删除元素
	- smembers key获取集合中所有元素
	- sismember key member判断元素是否在集合中
	- scard key获取集合中元素个数
	- srandmember key [数字] 从集合中随机弹出一个元素，元素不删除
	- spop key [数字] 从集合中随机弹出一个元素，出一个删除一个
	- 集合运算
		- 差集运算：sdiff key [key...]
		- 交集运算：sinter key [key...]
		- 并集运算：sunion key [key...]
- zset（sorted set）
	- zadd key score member [score member...]添加元素
	- zrange key start stop [withscores] 按照分数从小到大返回索引从start至stop之间的所有元素
	- zscore key  member 获取元素的分数
	- zrem key member [member] 删除元素
	- zrangebyscore key min max [withscores] [limit offset count] 获取指定分数范围的元素
	- zincrby key incremet member 增加某个元素的分数
	- zcard key 获取集合中元素的数量
	- zcount key min max 获取指定分数范围内的元素个数
	- zremrangebyrank key start stop 按照排名范围删除元素
	- 获取元素的排名：
		- 从小到大：zrank key member
		- 从大到小：zrevrank key member
- bitmap
- HyperLogLog
- GEO
- Stream

**redis的命令不区分大小写，但是key是区分大小写的**
## 命令
- help 例如help@string获取String类型的操作命令列表

## 持久化
**RDB和AOF可以同时存在；同时存在的时候找aof备份**
### RDB （Redis Database）
在指定的时间间隔内将内存中多的数据集快照写入磁盘，也就是行话说的Snapshot快照，它恢复时将快照文件直接读入内存

redis是单独创建（fork）一个进程来进行持久化，先将数据写入到一个临时文件中，等持久化过程都结束了，再用这个临时文件替换上次持久化好的文件。整个过程中，主进程不进行任何IO操作，这就保证了极高的性能，如果需要进行大规模的数据恢复，且对于数据恢复的完整性不是非常敏感，那么RDB方式要比AOF更加高效，RDB的确定是最后一次持久化后的数据可能丢失

fork：Fork的作用是复制一个与当前线程一样的进程，新进程的所有数据（变量、环境变量、程序计数器等）数值都是和袁金成一样的，但是是一个全新的进程，并作为原进程的子进程

文件名：dump.rdb；通过dbfilename dump.rdb配置文件 的名称
配置：
使用save设置备份策略：
save ""关闭
save 900 1 表示900秒内有一次修改就进行RDB
save 300 10 表示300秒内有10次修改就进行RDB
save 60 10000 表示60秒内有10000次修改就进行RDB
**备份文件dump.rdb直接覆盖，所以需要定时备份这个文件**

**
关闭服务（shutdown）或者flushall会清空所有key，同时RDB也会进行一次备份，文件是空的，所以手动关闭之前需要先备份一下dump.rdb
**

#### 备份时机
- 根据save的配置自动进行
- 使用save命令立即进行，这时候其他操作全部阻塞；只进行备份
- 使用bgsave命令：后台异步快照；快照的同时还可以响应客户端的请求，可以通过lastsave命令获取最后一次成功执行快照的时间
- 执行flushall，也会产生dump.rdb文件，但是是空的
#### 关闭RDB备份
在配置文件中设置：save ""
动态修改配置：config set save ""
#### 相关参数：
- stop-writes-on-bgsave-error yes：表示后台save备份的时候出错，前台停止写操作；如果配置成no表示不在乎数据一致性，或者有其他手段控制
- rdbcompression yes：启动LZF压缩算法；如果不想耗费CPU资源进行压缩，可以关闭
- rdbchecksum yes：存储快照后，让redis使用CRC64算法来进行数据校验，但是这样做大约会增加10%的性能消耗，如果希望获取最大的性能提升，可以关闭这个功能

#### 优劣势

适合大规模的数据恢复
对数据完整性和一致性要求不高

在一定能够间隔时间做一次备份，所以如果redis意外down掉，就会都市最后一次快照后的修改
Fork的时候，内存的数据被克隆了一份；

### AOF （Append Only File）
以日志的形式来记录每个写操作，将Redis执行过得所有写指令记录下来（读操作不记录）；只能追加文件，不能改写文件，redis启动的时候，会读取该文件重构数据

文件名：appendonly.aof文件
配置：AOF默认是关闭的
appendonly no


#### 相关参数
appendfilename "appendonly.aof"设置备份文件名称
##### appendfsync
默认：**everysec**：每秒，异步操作，每秒记录，但是一秒内宕机，有数据丢失
**always**：同步持久化，每发生一次修改卢克记录到磁盘，性能差，数据完整性好
**no**：fsync是一个命令，在大多数的Linux系统下，每30s就会执行一次fsync；appendfsync no其实是交给linux服务器

#### rewrite
由于文件会越写越大，为了避免这种情况，新增了重写机制，当AOF文件的大小超过了所设定的阈值时，redis就会启动AOF内容压缩，只保留可以恢复数据的最小指令集，可以使用命令bgrewriteaof来手动压缩
##### 原理
AOF文件持续增长过大时，会frok一个进程来讲文件重写，也是先重写文件最后再rename，遍历新进程的内存中数据，每条记录有一条set语句；**重写aof文件的操作，并没有读取旧的aof文件，而是将整个内存中的书库内容用命令的方式重新写了一个新的aof文件**，与快照类似

##### 触发机制
redis会记录上次重写时aof的大小，默认配置是AOF文件大小是上次rewrite或大小的一倍且文件大于64M的时候触发
auto-aof-rewrite-percentage 100：表示超过上次rewrite后文件大小的100%触发
auto-aof-rewrite-min-size 64mb：触发rewrite的最小文件大小，默认是64M

### 文件校验和修复：
#### redis-check-aof
redis-check-aof --fix AOF文件
#### redis-check-dump

## 分布式锁
分布式微服务架构，拆分后各个微服务之间为了避免冲突和数据故障而加入的一种锁
实现方案一般有三种：
redis
zookeeper
mysql
一般习惯于使用redis做


```
public static final String REDIS_LOCK = "order_lock";

@Autowired
private StringRedisTemplate stringRedisTemplate;

public String orderDispattcher(){
	String lockValue = UUID.randomUUID();
	try{
		//设置锁和过期时间使用一次操作，保证原子性
		Boolean lockFlag = stringRedisTemplate.opsForValue().setIfAbsent(REDIS_LOCK, lockValue, 10L, TimeUnit.SECONDS);
		if(!lockFlag){
			return "资源已被占用";
		}
		//deal order
	}finally{
		//v-1这里判断锁是否是自己的那把，否则可能因为处理时间过长，锁已经过期，而释放了其他业务线程/服务加的锁；但是这个操作不是原子的
		if(stringRedisTemplate.opsForValue().get(REDIS_LOCK).equalsIgnoreCase(value)){
			stringRedisTemplate.delete(REDIS_LOCK);
		}
	}
	//v-2使用redis事务来实现原子操作
	while(true){
		stringRedisTemplate.watch(REDIS_LOCK);
		if(stringRedisTemplate.opsForValue().get(REDIS_LOCK)
				.equalsIgnoreCase(value)){
			stringRedisTemplate.setEnableTransactionSupport(true);
			stringRedisTemplate.multi();
			stringRedisTemplate.delete(REDIS_LOCK);
			List<Object> list = stringRedisTemplate.exec();
			if(list == null){//锁在监视过程中发生了改变
				continue;
			}
		}
		stringRedisTemplate.unwatch(REDIS_LOCK);
		break;
	}
	//v3使用lua脚本实现
	Jedis jedis = RedisUtils.getJedis();
	String script = "if redis.call('get', KEYS[1] == ARGV[1])"
					+ "then"
					+ "return redis.call('del', KEYS[1])"
					+ "else"
					+ "return 0"
					+ "end";
	try{
		Object 0 = jedis.eval(script, 
				Collections.singletonList(REDIS_LOCK), 
				Collections.singletonList(value));
		if("1".equals){
			//操作成功后的操作
		}
	}finally{
		if(jedis != null){
			jedis.close();
		}
	}
}
```
### 业务执行时间超过过期时间，如何进行缓存续命？
### AP模式下，redis异步复制造成锁丢失，如何解决？
比如主节点没有来得及将刚刚set进来的数据给从节点，就挂了，在主从模式下，从节点上位主节点，而其中并没有刚刚的数据
**Zookeeper是CP，可以保证数据一致**

### 使用Redisson

```
public static final String REDIS_LOCK = "order_lock";

@Autowired
private StringRedisTemplate stringRedisTemplate;
@Autowired
private Redisson redisson;

public string xxx(){
	RLock redissonLock = redisson.getLock();
	redissonLock.lock();
	try{
		//dosomething
	}finally{
		redissonLock.unlock();
	}
}
```
在超高并发的情况下，可能会出现异常IllegalMonitorStateException：attempt to unlock lock,not locked by current thread by node id:xxx
#### 异常解决
```
if(redisson.isLoked()){//判断是否是锁定状态
	if(redissonisHeldByCurrentThread()){//判断持有锁的线程是否是当前线程
		redisson.unLock();
	}
}
```

### 常见题目

#### 生产上你们的redis内存设置多少？
配置文件中maxmemory参数
默认这个参数是没有配置的，64位操作系统是不限制，32位最大是3G
**一般设置最大内存占用物理内存的3/4**
#### 如何配置。修改redis内存的大小？
- 修改配置文件
- 通过命令来配置，在运行过程中动态的修改
	- config get maxmemory 获取当前内存大小
	- config set maxmemory xxx 设置最大内存

使用info命令查看所有参数，加参数可以选定查看范围，比如：
info memory查看内存相关的参数，包括当前内存的使用情况
#### 如果内存满了怎么处理
报错OOM command not allowed when used memory > 'maxmemory'
#### redis清理内存的方式？定期删除和惰性删除了解过吗？

#### redis缓存淘汰策略

- 定时删除：到达指定时间，立即删除，会对CPU造成压力
- 惰性删除：数据到大过期时间，不做处理，等待下次访问该数据的时候，如果未过期，返回数据；如果已过期，删除，返回不存在；对内存不友好
	- 如果大量过期键，长时间没有访问到，就会有大量无用数据占用内存
- 定期删除：每隔一段时间进行一次过期键删除操作，并通过限制删除操作执行的时长和频率来减少删除操作对CPU时间的影响
	- 周期性轮询redis中的时效性数据，采用随机抽取的策略，利用过期数据占比的方式控制删除频度；1. CPU性能占用设置有峰值，检测频度可以自定义设置；2. 内存压力不是很大，长期占用内存的冷数据会被持续清理；**随机抽查，还是有可能存在过期键从未被抽查到，也没有访问，就会造成过期数据占用内存**

未解决上面问题，使用过期策略来实现：
- volatile- lru ：对设置了过期时间的key使用lru算法
- allkeys-lru ：对所有keys使用lru算法（一般情况下使用）
- volatile- lfu ：对设置了过期时间的key使用lfu算法
- allkeys- lfu：对所有keys使用lfu算法
- volatile- random：对设置了过期时间的key使用random算法
- allkeys- random：对所有keys使用random算法
- volatile- ttl：删除马上要过期的key
- noeviction  默认（不再驱逐）maxmemory- policy neoviction
**算法说明**
- LRU = Least Recently Used 近期最少使用
- LFU = Least Frequently Used 最少使用
- random = 随机删除
**配置**
在配置文件中配置：maxmemory- policy allkeys- lru
在命令行中配置：config set maxmemory- policy allkeys- lru

#### redis的LRU了解过吗？手写一个LUR算法
Least Recently Used 近期最少使用，一一种常见的页面置换算法，选择最近最久未使用的数据进行淘汰

lru的本质是使用hash+（双向）链表实现，这样才能够满足查找块
##### 使用linkedHashMap
```
public class LRUCache<K, V> extends LinkedHashMap<K, V> {
    //缓存位
    private int capacity;

    public LRUCache(int capacity){
        /**
         * access order
         * true：接近顺序
         * false：插入顺序
         */
        super(capacity, 0.75F, true);
        this.capacity = capacity;
    }
}
```
##### 不依赖jdk

## redis事务
可以执行多个命令，本质是一组命令的集合，一个事务中的所有命令都会被序列化，按照顺序执行，而不会被其他命令插入
### 常用命令：
- discard：取消事务，放弃执行事务块内的所有命令
- exec：执行所有事务块内的命令
- multi：取消watch命令对所有key的监视
- watch key [key ...]：监视一个或者多个key。如果在执行事务之前这个key被其他命令所改动，那么事务将被打断

```
watch a
multi
set a 10
set b 20
exec
返回ok成功。返回nil表示监视目标被修改
exec执行后，之前加的监控都会被取消（unwatch）
```

### 对事物的支持度
部分支持
比如：使用sadd添加元素到集合中，如果对应的key存储的不是集合，就会执行出错，但是事务中的其他命令会执行成功，只有这个返回错误

## redis发布订阅

命令：
- psubscribe pattem [pattem ...]：订阅一个或者多个通道（以通配符的形式）
- pubsub subcommand [argument [argument ...]]
- publish channel message
- subscribe channel [channel...] 订阅一个或者多个通道

## redis主从复制
Master以写为主，Slave以写为主；slave不可以进行写操作

- 读写分离
- 容灾恢复
info replication命令查看主从信息
### 配置和使用
配置从库不配置主库
配置两个三个节点一一相连，这样主节点挂了，它的子节点自动上位，但是第一个节点正常服务后就变成一个独立的节点了
- 从库配置：slaveof 主库IP 主库端口，这样配置后，每次与master断开连接口，都需要重新连接
- 在配置文件中slaveof 127.0.0.1 6380，如果有密码masterauth <master-password>

### 哨兵模式
https://www.cnblogs.com/imcati/p/10487445.html
https://blog.csdn.net/zsx18273117003/article/details/84342272
1. 创建sentinel.conf文件
2. 配置信息：

- 连接信息：```sentinel monitor 节点名字（自己起） ip:port 1 ```
	- 例如：```sentinel monitor mymaster 192.168.0.5 6379 2```
- 密码：```sentinel auth-pass <master-name> <password> ```
	- 例如：```sentinel auth-pass mymaster 0123passw0rd```
**设置连接master和slave时的密码，注意的是sentinel不能分别为master和slave设置不同的密码，因此master和slave的密码应该设置相同**
最后的参数：1，代表有几个哨兵认为master失效的时候，master就真正的失效
3. 启动sentinel服务redis-sentinel ../sentinel.conf

master挂掉后，哨兵进行选举，选出新的master；如果挂掉的节点重新正常，就会作为slave进行服务





