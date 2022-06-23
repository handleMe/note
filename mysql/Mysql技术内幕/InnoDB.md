# INNODB

## 体系架构
包括多个后台进程/线程、内存池、文件组成

### 后台线程
- 主线程：Master Thread，负责将缓冲池中的数据异步刷新到磁盘，保证数据一致性，包括脏页的刷新、合并插入缓存、UNDO页的回收等；
- IO Thread：InnoDB大量使用了AIO来处理IO，这样可以提升数据库性能，IO Thread负责这些IO请求的回调处理；在1.0.X+的版本，read thread和write thread分别有4个，可以使用innodb_read_io_threads和innodb_write_io_threads参数来进行设置；可以通过show engine innodb status来查看IO Thread
- Purge Thread：清除线程；事务提交后，其使用的undolog可能不再需要；此线程回收已经使用并分配的undo页；可以再配置文件中通过innodb_purge_threads=4来设置线程个数
- Page Cleaner Thread：1.2.x中引入，作用是将之前版本中脏页的刷新操作都放入单独的线程中来完成，减轻主线程压力

### 内存

#### 缓冲池
	数据库进行读取页操作，首先判断页是否在缓存池中，否则读取磁盘上的页，将从磁盘读到的页存储（FIX）到缓存池中；对于修改操作，首先修改在缓存池的页，然后以一定的频率刷新到磁盘中；刷新操作并不是每次页发生变更的时候触发，二是通过Checkpoint机制刷新回磁盘
	缓冲池大小直接影响数据库整体性能；可以通过参数innodb_buffer_pool_size=16106127360来设置，这里是15G
	具体看，缓冲吃中缓存的数据类型有：索引页、数据页、undo页、插入缓冲、自适应hash索引、Innodb存储的锁信息、数据字典信息等；
	1.0.X版本后允许有多个缓冲池实例，可以通过innodb_buffer_pool_instances来设置，默认是1

##### LRU List、Free List和Flush List
	缓冲池是通过LRU算法来进行管理的；最少使用的页在LRU列表的尾端；InnoDB的缓冲池中页的大小默认是16KB，在InnoDB中，缓冲池将新读取到的页存放在midpoint位置，在LRU列表长度的5/8处；midpoint可以通过参数来控制，innodb_old_blocks_pct=37；代表新页存储在LRU列表尾端的37%处；
	midpoint之前的列表称为new，后边称为old
	数据库刚启动时，LRU列表是空的，页都放在Free列表中，页从LRU列表的old部分加入到new部分时，称为**page made young**；因innodb_old_blocks_time设置导致页没有从old部分移动到new部分的操作称为**page not made young**
	为什么不直接将读取到的页放入LRU列表的首部呢？因为有些查询需要扫描很多页，这样的数据可能只是在本次操作中需要，而不是热点数据，这样可能将真正的热点数据从LRU列表中移除
	通过show engine innodb status来查看LRU列表及free列表的使用情况和运行状态
	Buffer pool hit rate表示缓冲池命中率
	自适应hash索引、lock信息、insert buffer等页不需要LRU算法进行维护，所以不存在于LRU列表中
	Flush列表：Modified db pages显示脏页的数量

#### 重做日志缓冲
	InnoDB首先将重做日志缓冲放入这个缓冲区，按照一定频率（一般是每秒）刷新到重做日志文件；因此只需要保证每秒产生的事务量在缓冲区大小之内即可；大小由innodb_log_buffer控制，默认是8M 8388606
##### 以下情况会将重做日志缓存刷入日志文件：
- Master Thread每一秒将日志缓存刷入重做日志文件
- 每个事务提交时会将重做日志缓存刷入到重做日志文件
- 当重做日志缓冲池剩余空间小于1/2时，刷入

#### 额外的内存池
	在InnoDB中，对一些数据结构本身的内存进行分配时，需要从额外的内存池中进行申请，该区域内存不足时，会从缓冲池中进行申请；在申请了很大的缓冲池的时候，也需要考虑相应的增加这个值；

### checkpoint技术
	一条DML语句修改页，这个记录会先存在缓冲池中，此时页时脏的（即：缓冲池中的数据比磁盘新）；
	每次修改都将数据刷新到磁盘，开销非常大；但如果刷入数据的时候发生了宕机，数据就不能恢复了。为避免这种情况，采用Write Ahead Log策略，即事务提交时，先写重做日志，再修改页；宕机时，可以通过重做日志来恢复。这也是持久性的要求
#### checkpiont解决的问题：
- 缩短数据库恢复时间
- 缓冲池不够用的时候，将脏页刷新到磁盘
- 重做日志不可用时，刷新脏页

#### 版本标记LSN
log sequence number；每个页中有LSN，重做日志中也有LSN，checkpoint中也有LSN

**show engine innodb status**查看LSN

```
---
LOG
---
Log sequence number xxxxxxx
Log flushed up to xxxxxx
Last checkpoint at xxxxxx
```
#### 两种checkpoint

##### Sharp Checkpoint
刷新所有的脏页
发生数据库关闭时，默认的方式，innodb_fast_shutdown=1
但如果数据库运行时页使用Share Checkpoint，数据库性能会受到影响；所以InnoDB内部使用Fuzzy Checkpoint刷新，只刷新一部分脏页，而不是所有

##### Fuzzy Checkpoint
只刷新一部分脏页，而不是所有；以下几种情况使用
- Master Thread Checkpoint 每秒或每十秒发生，异步
- FLUSH_LRU_LIST Checkpoint 因为需要保证LRU有100个空闲页可使用而进行；1.2.x版本后，这个检查在Page Cleaner线程中进行，可以通过innodb_lru_scan_depth控制表中可用页数量，默认1024
- Async/Sync Flush Checkpoint 重做日志不可用的情况下，刷入；
- Dirty Page too much Checkpoint 脏页太多强制进行checkpoint；innodb_max_dirty_pages_pct=75表示脏页占据缓冲池75%的时候，刷新一部分脏页到磁盘；默认75

## Master Thread的工作方式

### 1.0.x版本之前的工作方式
	具有最高的优先级别，内部由多个循环组成，主循环loop、后台循环background loop、刷新循环flush loop、暂停循环suspend loop；
	主线程会根据数据库状态在这些循环中切换；执行每秒钟或者每十秒的操作
	通过thread sleep来实现每秒或每十秒进行操作，意味着并不准确；负载很大的时候可能会有延迟

#### 每秒的操作：
- 日志缓冲刷新到磁盘，即使这个事务还没有提交（总是）；因为总是执行，所以大事务的commit时间也会很短
- 合并插入缓存（可能）；当前一秒IO次数小于5，认为IO压力很小，可以执行合并插入缓存的操作
- 最多刷新100个innodb的缓冲池中的脏页到磁盘（可能）；见上一个条目
- 如果当前没有用户活动，切换到background loop（可能）

#### 每十秒的操作
- 刷新100个脏页到磁盘（可能）；过去10秒内IO操作次数小于200
- 合并最多5个插入缓存（总是）；
- 将日志缓冲刷新到磁盘（总是）
- 删除无用的undo页（总是）；full purge；每次最多尝试回收20个undo页
- 刷新100个或者10个脏页到磁盘（总是）；缓冲池中有超过70%的脏页，刷新100个到磁盘，小于70%，刷新10%的脏页到磁盘


#### background loop
数据库空闲（没有用户活动）或者数据库关闭的时候，会切换到这个循环
- 删除无用的undo页（总是）；
- 合并20个插入缓冲（总是）；
- 跳回主循环（总是）；
- 不断刷新100个页直到符合条件（可能，跳转到flush loop中完成）；flush loop中没有事情可以做，就切换到 suspend loop，将Master Thread挂起，等待事件

如果用户启用了InnoDB存储引擎，但是没有任何表使用InnoDB，那Master Thread总是处于挂起的状态

### 1.2.x之前的修改
提供了参数innodb_io_capacity，用来表示磁盘IO的吞吐量，默认值200
- 在合并插入缓冲时，合并插入缓冲的数量为innodb_io_capacity的5%
- 在从缓冲区刷新脏页时，刷新脏页的数量为innodb_io_capacity

提供了参数innodb_adaptive_flushing（自适应刷新），影响每秒刷新脏页的数量

提供了参数innodb_purge_batch_size，控制每次full purge回收的undo页数量，默认是20

### 1.2.X版本及之后
刷新脏页的操作分离到线程Purge Cleaner Thread中，减轻了Master Thread的压力

## 关键特性
- 插入缓冲 Insert Buffer
- 两次写 Double Write
- 自适应hash索引 Adaptive Hash Index
- 异步IO Async IO
- 刷新邻接页 Flush Neighbor Page

### 插入缓冲
	对于非聚集索引来说，插入和修改操作不是直接插入到索引中的，二是先判断插入的非聚集索引页是否在缓冲池中，若在，直接插入；不再，则先放在一个Insert Buffer对象中；然后以一定的频率和情况进行Insert Buffer和辅助索引叶子结点的merge操作，这时通常能将多个插入合并到一个操作中，这样就提高了非聚集索引插入的性能。
需要满足两个条件：
	- 索引是二级索引
	- 索引不是唯一的；插入缓存时，不需要去查询判断是否有唯一性冲突，如果是唯一键，就需要去查找判断，这就失去了使用缓冲的意义
可以通过show engine innodb status来查看插入缓冲的信息：

```java
...
Ibuf: size xxxx, free list len xxxx, seg size xxxx,
xxxxx inserts, xxxxx merged recs, xxxxx merges
...
seg size 显示了当前Insert Buffer大小为value*16KB
free list size 代表空闲列表长度
size表示已经合并记录页的数量
```
写操作密集的情况下，插入缓冲会缓冲池较多空间（默认最多1/2）
可以通过参数ibuf_pool_size_per_max_size=3设置，3表示最大只能使用1/3

#### change buffer
	1.0.x版本开始，引入了Change Buffer，可以视为是Insert Buffer的升级，InnoDB对DML分别进行缓冲，Insert Buffer、Delete Buffer、Purge Buffer
	和之前一样，适用于非唯一的二级索引
对一条记录的删除可以分为两个过程：
- 将记录标记为已删除
- 真正的删除
Delete Buffer对应第一步，Purge Buffer对应第二步
##### 参数
- innodb_change_buffering 用来开启各种Buffer选项
可选值：inserts、deletes、purges、changes、all、none
changes表示inserts和deletes
all表示所有
none表示不启用
默认为all

- innodb_change_buffer_max_size=25来控制Change Buffer最大使用比例（占缓冲池）
最大有效值50；25表示25%

##### 结构
Insert Buffer的结构是一颗B+树；在4.1之前，每张表都有一颗Insert Buffer的B+树，现在全局只有一颗B+ 树

这个数据结构存放在共享表空间中，也就是ibdata1中，因此通过独立表空间ibd文件恢复表中数据的时候，可能会导致check table失败，这是因为二级索引的数据可能还在insert buffer中；所以通过ibd文件进行恢复后，还需要进行repair table操作来重建表上的辅助索引

非叶子节点存储的是查询的search key，共9个字节
- space：表示待插入记录所在表的表空间id，4字节
- marker：占用1字节：用来兼容老版本的Insert Buffer， 1字节
- offset：表示页所在的偏移量，4字节

叶子节点存储：
space、marker、offset、metadata、secondary index record
前三个与飞叶子节点意义相同，占用9个字节
metadata占用4个字节：
	- IBUF_REC_OFFSET_COUNT：2字节整数，排序每个记录进入Insert Buffer的顺序
	- IBUF_REC_OFFSET_TYPE：1字节
	- IBUF_REC_OFFSET_FLAGS：1字节
后面是实际插入记录的各个字段

	为了保证每次merge Insert Buffer页必须成功，需要有一个特殊页来标记每个辅助索引页的可用空间；这个页的类型为Insert Buffer Bitmap
Insert Buffer Bitmap构成：
- IBUF_BITMAP_FREE：2bit；0表示无剩余空间、1表示剩余空间大于512字节、2表示大于1KB、3表示大于2KB
- IBUF_BITMAP_BUFFERED：1bit，1表示该辅助索引页有记录缓存在Insert Buffer B+树种
- IBUF_BITMAP_IBUF：1bit，1表示该页为Insert Buffer B+树的索引页

#### Merge Insert Buffer
Insert Buffer如何进行合并？

发生情况：
- 辅助索引页被读取到缓冲池时
- Insert Buffer Bitmap页追踪该辅助索引页已无可用空间时；
- Master Thread

### 两次写
	如果页在写入的时候宕机，就会出现部分写失效
	doublewrite：提高数据页的可靠性；由内存中的doublewrite buffer（约2MB）和物理磁盘的共享表空间ibdata中连续的128的页（2个extent）构成；
	过程：脏页刷新的时候，先复制到doublewrite buffer，之后通过doublewrite buffer再分两次，每次1MB顺序的写入到共享表空间，然后调用fsync函数，同步磁盘；
	skip_innodb_doublewrite可以紧致使用doublewrite功能，这可能会发生部分写失效，所以在需要提供数据高可靠性的主服务器，还是需要开启双写的

### 自适应hash索引 AHI adaptive hash index
	InnoDB会根据访问频率和模式自动为**某些热点页**建立hash索引
	AHI通过缓冲池的B+树页构造出来，建立速度很快，不需要对整张表建立；
AHI建立的条件：
	以某种访问模式连续进行访问（访问模式相同指查询条件一样）
	- 以一种模式访问了100次
	- 页通过一种模式访问了N次，N=页中记录/16

```
show engine innodb status
------------------
INSERT BUFFER AND ADAPTIVE HASH INDEX
--------------------
...
hash table size xxxxxx, non heap has xxx buffer(s)
xxxx hash searches/s, xxxxx non-hash searches/s
...
```
innodb_adaptive_hash_index 禁用或开启自适应hash索引，默认开启

### 异步IO
	AIO实现依赖libaio；windows和linux默认提供支持，mac OSX则没有提供
	AIO有异步的优势，速度快；还可以进行IO merge，即将多个IO合并为一个IO；例如：需要访问页（space， page_no）为：（8,6）、（8,7），AIO判断这两个页是连续的，会合并成一个IO，从（8,6）开始，读取32KB的页数据
	参数innodb_use_native_aio可以控制是否启用Native AIO；linux系统下时默认开启的
	在InnoDB中read ahead方式的读取都是通过AIO完成，脏页的刷新，即磁盘的写入操作全部由AIO完成
### 刷新邻接页
	当刷新一个脏页的时候，InnoDB检测到该页所在区（extent）的所有页，如果是脏页，就一并进行刷新；这样就可以将多个AIO合并成一个写操作
	- 是不是将脏页刷入后，这个页很快又被修改，变成脏页
	- 固态硬盘具有较高的IOPS，是否需要这个特性
可以通过innodb_flush_neighbors；用来控制是否开启邻接页刷新；固态硬盘建议设置为0，即关闭

## 启动、关闭与恢复

### 关闭
参数innodb_fast_shutdown影响着表存储引擎为InnoDB时的行为；默认值1
- 0：表示MySQL数据库关闭时，InnoDB需要完成所有的full purge和merge insert buffer；并且将所有的脏页刷新回磁盘。如果实在进行InnoDB升级的时候，必须将这个参数调为0，然后关闭数据库
- 1：是默认值，表示不需要完成full purge和merge insert buffer操作，但是在缓冲池的一些脏数据页还是会刷新回磁盘
- 2：表示不完成full purge和merge insert buffer操作，也不刷新buffer中的脏页；二是将日志都写入日志文件。这样不会有任何事务丢失，下次启动时，也会进行恢复操作。

inndo_force_recovery：影响存储引擎进行恢复的状况；默认是0
- 0：默认值，代表发生需要恢复情况时，进行所有的而恢复操作，不能恢复时，数据库可能宕机（crash），并把错误写入日志
- 1：SRV_FORCEIGNORE_CORRUPT：忽略检查到的corrupt（损坏）页
- 2：SRV_FORE_NO_BACKGROUND：阻止Master Thread线程的运行，阻止如：Master Thread需要进行full purge操作，会导致crash
- 3：SRV_FORCE_NO_FIX_TRX_UNDO：不进行事务的回滚操作
- 4：SRV_FORCE_NO_IBUF_MERGE：不进行插入缓冲的合并操作
- 5：SRV_FORCE_NO_UNDO_LOG_SCAN：不查看撤销日志（undo log），InnoDB存储引擎会将未提交事务视为已提交
- 6：SRV_FORCE_NO_LOG_REDO：不进行前滚操作

**inndo_force_recovery设置参数不是0，用户不能进行DML操作（insert、update、delete等）；select、create和drop操作是允许的**


