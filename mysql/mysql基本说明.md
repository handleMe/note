## 准备阶段
### 局部性原理：
经常被查询的数据具有聚集成群的倾向，同时，刚刚被查询的数据有可能会很快被再次查询。

### 磁盘预读：
磁盘和内存进行交互的最小逻辑单元，和操作系统有关，一般是4K或者8K，可以称为一页（datapage）

### 数据文件
```
MyISAM存储文件分3个，
test_myisam.frm 存储表结构
test_myisam.MYD 存储表数据
test_myisam.MYI 存储索引数据

InnoDB存储文件分2个
test_innodb.frm 存储表结构
test_innodb.ibd 存储表数据和索引 表结构本身就是按照B+tree组织的一个索引结构文件
```
```
客户端
	连接器：管理连接、校验权限
	缓存区：cache查询结果
	词法分析器：词法、语法分析
	优化器：执行计划生产、索引选择
	执行器：调用引擎接口、获取查询结果
	存储引擎：
```

### Mysql执行查询的过程：
连接服务器，经过连接器校验后连接，发送指令，词法分析器进行分析，优化器优化语句病进行索引选择，调用执行引擎，在存储引擎中查询结果，如果开启缓存，将结果存入缓存区，返回执行结果
- 客户端发送一套查询给服务器
- 检查缓存，命中直接返回
- 进行SQL解析、预处理，交由优化器生成执行计划
- 根据执行计划，调用存储引擎API进行查询
- 返回结果

## 缓存
Mysql缓存机制：
- 缓存数据以k-v形式
- 查询缓存在5.8版本被移除
- 适用于读多写少的场景
- 需要在配置文件中指定参数query_cache_type为（0,1,2）关闭、开启、按需使用
- 按需使用：select SQL_CACHE * from tablename;指定参数为2，加入SQL_CACHE关键字，这条sql就会使用缓存
- 查询cache参数：show status like '%Qcache%'

## Bin Log归档日志：
在mysql的server层实现的，所有引擎公有的，Bin Log是逻辑日志，记录的是一条语句的原始逻辑，没有大小限制，追加写入，不会覆盖以前的日志

查看bin log的启用状态：show variables like '%log_bin%';
flush logs；刷新日志存储，使用一个新的日志文件来记录操作；

1.binlog文件会随服务的启动创建一个新文件

2.通过flush logs 可以手动刷新日志，生成一个新的binlog文件

3.通过show master status 可以查看binlog的状态

4.通过reset master 可以清空binlog日志文件

5.通过mysqlbinlog 工具可以查看binlog日志的内容

6.通过执行dml，mysql会自动记录binlog

end_log_pos 表示一个操作

Redo Log重做日志：仅InnoDB支持
	
	

## B+ Tree和B树的区别：


## 数据类型及优化
- 更小的通常更好：一般情况下，尽量使用可以正确存储数据的最小数据类型，更小的通常更快，因为占用更少的磁盘、内存和CPU缓存，并且处理时需要的CPU周期也更少
- 使用更简单的类型，例如不要使用字符串来存储日期
- 尽量避免null，查询中存在可能为null的值，对MySQL来说更难优化，因为可为null的使得索引、索引计算和值比较变得复杂；如果计划在列上建索引，就需要尽量避免设计成可为null

### varchar和char类型
**字符串长度定义的是字符数而不是字节数**
#### varchar
 varchar是变长的，相比于定长存储的类型节省了空间，但是需要1到2个字节存储长度（小于等于255字节1个字节，否则两个字节）；
但是由于长度的不固定，导致update的时候需要额外的工作；如果修改时在页内没有更多的空间增长，MyISM会拆成不同的片段存储，InnoDB则需要分裂页来使行可以放进页内。

##### 下面两种情况适合使用varchar进行存储
- 字符串的最大长度较平均长度大得多
- 列的更新很少，所以碎片不是问题
- 使用了UTF8这样的字符集，字符可能使用不同的字节进行存储

#### char
char是定长的，**存储时会删除末尾的空格，慎用**，适合存储例如密码的加密值等定长的值；对于变长的值，varchar更适合

**填充和截取空格的行为对不同的存储引擎是一样的，因为这是在服务层进行的，而不是在存储引擎层进行的**

### BLOB和TEXT类型
MySQL把每个BLOB和TEXT值当做一个单独的对象处理，值太大时，InnoDB会使用专门的“外部”空间进行存储，此时每个值在行内需要1~4个字节存储一个指针，在外部存储实际的值

排序：对 BLOB和TEXT类型来说，MySQL只对前max_sort_length的字节做排序，可以通过设置这个配置值来控制排序有效的长度

磁盘临时表：查询这两个数据类型可能会使用MyISAMySQL磁盘临时表，即使只有几行数据也是如此，**所以尽量避免使用BLOB和TEXT**，可以使用substring函数将类型转换为字符串，这样就可以使用内存临时表了，但是长度超过max_heap_table_size或者tmp_table_size，还是会使用MyISAMySQL磁盘临时表**explain中之心国际化中包含了Useing temporary，说明这个查询使用了隐式临时表**

### ENUM枚举
枚举列可以把字符串使用预定义的集合进行存储，在.frm中存储“数字-字符串的映射关系”

排序：枚举列是按照内部存储的整数进行排序的，而不是字符串值，可以使用field函数指定排序顺序，但是这样无法使用索引

缺点：字符串列表是固定的，要修改就需要alter table；


### 日期时间类型

#### DATETIME
保存范围从1001年到9999年，精度是秒，使用整数存储，占用8个字节

#### TIMESTAMP
保存范围从1970年1月1日志2038年，精度是秒，使用整数存储，占用4个字节

### 主键列的选择
整数列是最好的选择，因为他们很快并且可以使用自增
避免适应字符串作为主键，很消耗空间，对于随机的字符串如MD5或者UUID，这些值插入时会随机的写入索引的不同位置，使得insert语句变慢；也会使得缓存赖以工作的局部性原理失效，缓存会有很多刷新和未命中

### schema设计的陷阱

- 太多的列，MySQL的存储引擎和服务层之间通过缓冲格式拷贝数据，在服务器层将缓冲内容解码成各个列，列的个数越多，操作代价越高
- 太多的关联，关联查询太多会导致解析和优化的代价变大
- 枚举：修改枚举范围需要alter table
- 不要使用set来替代enum
- 不要使用不可能的值作为默认值，例如使用“0000-00-00”来作为日期的默认值

### 范式和反范式
范式化的表，更新快，重复数据少或者没有，修改时需要修改的地方少，检索时也更少的使用distinct或者group by

反范式的表设计，数据都在少数的表中，冗余较多，但是可以减少表关联，这样避免了随机I/O

**混用范式和反范式化的表设计**

### 缓存表和汇总表
保存衍生的冗余数据是提升性能的一种方式，比如会用一张定时统计数据的汇总表，牺牲一定的数据实时性，提升统计的效率；再比需要从数张表中联合查询信息，可以将这些信息冗余到另一张表中，牺牲存储空间来提升查询效率

### 计数表
例如表示用户好友数、文件下载次数等
在表中保存计数器，可能会遇到大量并发的问题，因为针对某个目标的的计数器只有一条数据，可以拆分为多条数据，比如拆成10条数据，更新的时候使用随机函数更新某一条，以降低写入并发量

### ALTER TABLE
Mysql的Alter table有可能引起表重建，性能比较低，大部分修改表结构的操作方式时用新的结构创建一个空表，在从旧表查出数据插入新表
下面的情况是不会重建表的
- 移除一个列的自增
- 增加、移除或者更改ENUM和SET的常量，移除后，如果数据中还有这个常量对应的值，查询会返回空串
- 使用alter table table_name alter column field set default value；而不是使用 alter table 表名 modify column 字段 ...... default 值;

## 索引


聚集索引：叶子结点包含了完整地数据，innodb的主键索引就是这个，innodb非主键索引是非聚集索引，叶子节点存储的是索引值和主键
非聚集索引：叶子结点没有存储完整地数据，myisam的主键索引树和普通索引树存的内容都是当列的值和数据行地址（行号）

### 回表：
根据普通索引查询到id（或者聚簇索引的key），再去聚簇索引中查询数据，称为回表

### 索引覆盖：
已知表中有id'作为主键，name作为索引，select id,name from table where name='a';在name的索引中查询就可以得到id和name的值，就不需要回表，称为索引覆盖

### 最左匹配：
联合索引讲所有的字段值拼接在一起作为索引键来排序，最左前缀原理就是由此而来
为什么跳过索引前面的字段直接使用后面的字段作为条件不能使用索引？
因为跳过前面的字段，后面的字段就不是排好序的了，所以不能使用索引
```
有表test，以name，age作为联合索引
select * from test where name= '' and age = '';
select * from test where name= '';
select * from test where age = '';
select * from test where age = '' and  name= '';

其中124回使用索引，其中4会被mysql的优化器调整条件顺序来适配索引
```

### 索引下推：
索引下推（INDEX CONDITION PUSHDOWN，简称 ICP）是 MySQL 5.6 发布后针对扫描二级索引的一项优化改进。总的来说是通过把索引过滤条件下推到存储引擎，来减少 MySQL 存储引擎访问基表的次数以及 MySQL 服务层访问存储引擎的次数。ICP 适用于 MYISAM 和 INNODB
ICP 默认开启，可通过优化器开关参数关闭 ICP：optimizer_switch='index_condition_pushdown=off' 或者是在 SQL 层面通过 HINT 来关闭。
```
有表test，以name，age作为联合索引
select * from test where name= '' and age = '';
没有索引下推，根据name从存储引擎中获取符合规则的数据，然后再server层根据age进行过滤
有索引下推：根据name和age从存储引擎中获取数据
```

### 在排序中使用索引

- 只有索引的顺序和order by的顺序完全一致，并且排序方向都一样的时候（正序或者倒序），才能在排序时使用到这个索引；
- 多张表关联的时候，只有order by字句中引用的字段全部是第一张表的时候，才能使用索引排序
- 排序也需要满足最左匹配原则
```
索引（rental_date,inventory_id，customer_id）
查询：select * from table_name where rental_date='2020-01-01'
		order by inventory_id,customer_id
这个查询可以使用索引进行排序，因为rental_date被条件锁定为一个常量，满足了最左匹配，如果使用范围值，则不能使用到索引，比如>

下面的语句不能使用索引排序：where rental_date='2020-01-01' and inventory_id in(2,3) order by customer_id
因为in被视为是范围查找

以下情况不能使用索引排序：使用不同的排序方向、含有索引以外的排序字段、跳过了索引中得某个列
```
对于分页查询到较大页数的时候，可以考虑使用条件直接筛选到目标条目的开始，比如使用>某个时间或者>某个主键来减少返回的条数；
或者可以使用延时关联，即在子查询中查出目标条目的主键，然后关联自身查出所有的列
```
select a.* from table_name a inner join (
	select x.id from table_name x where x.type=1 order by x.rating limit 100000, 10
) as b on a.id = b.id
```

Hash索引：存储索引和数据磁盘文件地址；不支持范围查找

B+Tree：

为什么建议InnoDB表必须建主键，并且推荐使用整形的自增主键？
InnoDB必须使用B+树组织数据，如果没有主键，mysql就会在列中寻找一个数据是唯一的列进行组织，如果没有找到这样的列，就建一个隐藏列（rowid）来组织数据存储；



为什么非主键索引的叶子结点存储的是主键值？（一致性和节省存储空间）
###  碎片
行碎片：数据行被存储为多个地方的多个片段中，即使查询只从索引访问中就能查询到一行数据，行碎片也会导致性能下降
行间碎片：逻辑上的顺序页，或者行在磁盘上不是顺序存储的，行间碎片对全表扫描和聚簇索引扫描之类的操作有很大影响，因为这些操作能从磁盘上顺序存储的数据中获益
剩余空间碎片：数据页中有大量的剩余空间，这回导致服务器读取到大量不需要的数据，从而造成浪费
可以通过重建表或者导出导入来整理磁盘碎片



## 字符集：
 MySQL在5.5.3之后增加了这个utf8mb4的编码，mb4就是most bytes 4的意思，专门用来兼容四字节的unicode。好在utf8mb4是utf8的超集，除了将编码改为utf8mb4外不需要做其他转换。当然，为了节省空间，一般情况下使用utf8也就够了。
   内容描述既然utf8能够存下大部分中文汉字,那为什么还要使用utf8mb4呢? 原来mysql支持的 utf8 编码最大字符长度为 3 字节，如果遇到 4 字节的宽字符就会插入异常了。三个字节的 UTF-8 最大能编码的 Unicode 字符是 0xffff，也就是 Unicode 中的基本多文种平面(BMP)。也就是说，任何不在基本多文本平面的 Unicode字符，都无法使用 Mysql 的 utf8 字符集存储。包括 Emoji 表情(Emoji 是一种特殊的 Unicode 编码，常见于 ios 和 android 手机上)，和很多不常用的汉字，以及任何新增的 Unicode 字符等等
   
