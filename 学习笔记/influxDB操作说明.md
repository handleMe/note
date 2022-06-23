
# 一、数据库操作

- 1.1 查看当前数据库
```
show databases
```
** _internal是内置的数据库 **
- 1.2 创建数据库
```
create database testdb
```
- 1.3 使用数据库
```
use testdb
```
- 1.4 删除数据库
```
drop database testdb
```

# 二、  策略基本操作
## 2.1 策略操作
- 创建retention policy
<font color=follow>retention policy</font> 依托于database存在，需要指定具体的数据库
```
CREATE RETENTION POLICY <retention_policy_name> ON <database_name> DURATION <duration> REPLICATION <n> [SHARD DURATION <duration>] [DEFAULT]
```
- retention_policy_name: 策略名（自定义的）
- database_name: 一个必须存在的数据库名
- duration: 定义的数据保存时间，最低为1h，如果设置为0，表示数据持久不失效（默认的策略就是这样的）
- REPLICATION: 定义每个point保存的副本数，默认为1
- default: 表示将这个创建的保存策略设置为默认的
数据保存一年的策略：
```
create retention policy "1Y" on test duration 366d replication 1
```
## 2.2 策略查看
```
show retention policies on <database name>
```
## 2.3 策略修改
```
ALTER RETENTION POLICY <retention_policy_name> ON <database_name> DURATION <duration> REPLICATION <n> SHARD DURATION <duration> DEFAULT
```
case:
```
alter retention policy "1Y" on testdb duration 365d replicatioon 2 default
```
## 2.4 删除策略
```
DROP RETENTION POLICY <retention_policy_name> ON <database_name>
```

# measuerment操作
可以将measurement类比做表
## 查看measurement
```
SHOW MEASUREMENTS [ON <database_name>] [WITH MEASUREMENT <regular_expression>] [WHERE <tag_key> <operator> ['<tag_value>' | <regular_expression>]] [LIMIT_clause] [OFFSET_clause]
case：
show measurements
show measurements on test
show measurements on test with measurement =~ /yhh*/
```
## 创建measurement
measurement不需要专门来创建，在向某个measurement插入数据的时候，如果measurement不存在，就会创建一个
```
insert userInfo,name=一灰灰blog userId=10,blog="https://blog.hhui.top/"
> show measurements
name: measurements
name
----
doraemon
doraemon2
userInfo
```
## 删除measurement
两种方式可以删除
- 将mwasurement中的数据全部删除，measurement就不存在了
```
delete from userInfo where time=1563712849953792293
```
- 直接使用drop命令删除
```
drop measurement userInfo
```
## 修改
没有直接的修改操作，如果希望修改表名，可以使用select into将查询结果保存到另一张表中
```
select * into userBaseInfo from userInfo
```
# 字段说明
influxdb中的一条记录point，主要可以分为三类，必须存在的time（时间），string类型的tag，以及其他成员field；而series则是一个measurement中保存策略和tag集构成；
## tag
influxdb中记录元数据的kv对，不要求必须存在，tag key/value 都是字符串类型，而且会建立索引，因此给予tag进行查询效率比单纯的基于field进行查询是要高的；某些查询只能基于tag
<font color=red  size=4> 重点： tag key/value 字符串类型；；有索引 </font>
### 查询tag
```
show tag keys on <database> from <measurement>
例子：
> insert yhh,name=一灰灰 age=26,id=10,blog="http://blog.hhui.top"
> select * from yhh
name: yhh
time                age blog                 id name
----                --- ----                 -- ----
1563888301725811554 26  http://blog.hhui.top 10 一灰灰
> show tag keys from yhh
name: yhh
tagKey
------
name
```
### tag values使用
```
show tag values on <database> from <measurement> with KEY [ [<operator> "<tag_key>" | <regular_expression>] | [IN ("<tag_key1>","<tag_key2")]] [WHERE <tag_key> <operator> ['<tag_value>' | <regular_expression>]] [LIMIT_clause] [OFFSET_clause]
例子：
show tag values from currency_rate with key="base"

```
- with key 后面带上查询条件，必须存在，如查询汇率表中，base_symbol有哪些
- 连接符号可以为：等于 =, 不等于：!=, <>, 正则：=~, !~

## field
成员，也可以理解为一条记录中，不需要建立索引的数据，一般来说，不太会有参与查询语句建设的可以设置为field
区别与tag，field有下面几个特性：
- 类型可以是：浮点型、字符串、整型
- 没有建立索引
查看field key：
```
show field keys on <database> from <measurement>
例子：
> show field keys from yhh
name: yhh
fieldKey fieldType
-------- ---------
age      float
blog     string
id       float
```
## point
可以理解为是一条记录，由一下四部分组成：
- measurement
- tag set
- field set
- timestamp

每个point是根据 <font color=red> timestamp + series </font> 来保证唯一性。

## series
influxdb中measurement + tags set + retention policy 组成的数据集合
`show series on <database> from <measurement>`
采用几个示例说明一下：
```
先插入两条数据查看一下
> insert yhh,name=一灰灰 age=26,id=10,blog="http://blog.hhui.top"
> insert yhh,name=一灰灰 age=30,id=11,blog="http://blog.hhui.top"
> show series on test from yhh
key
---
yhh,name=一灰灰
```
可以看到series包含了measurement和tag set
新增一个tag看看
```
> insert yhh,name=一灰灰2 age=30,id=11,blog="http://blog.hhui.top"
> insert yhh,name=一灰灰3,phone=110 age=30,id=11,blog="http://blog.hhui.top"
> show series on test from yhh
key
---
yhh,name=一灰灰
yhh,name=一灰灰2
yhh,name=一灰灰3,phone=110
```
可以看到series发生了变化
series还与保存策略有关
```
> create retention policy "1D" on test duration 1d replication 1
> insert into "1D" yhh,name=一灰灰4 age=26,id=10,blog="http://blog.hhui.top"
> show series
key
---
yhh,name=一灰灰
yhh,name=一灰灰2
yhh,name=一灰灰3,phone=110
yhh,name=一灰灰4
```
插入到”1D”保存策略中的point也构成了一个series: yhh,name=一灰灰4
show series还支持基于tag的条件查询
```
show series from yhh where "name" = '一灰灰'
key
---
yhh,name=一灰灰
> show series from yhh where phone != ''
key
---
yhh,name=一灰灰3,phone=110
```
# inert使用说明
## 基本语法：
```
insert into <retention policy> measurement,tagKey=tagValue fieldKey=fieldValue timestamp
示例：
insert add_test,name=YiHui,phone=110 user_id=20,email="bangzewu@126.com"
```
- tag与tag之间用逗号分隔；field与field之间用逗号分隔
- tag与field之间用空格分隔
- tag都是string类型，不需要引号将value包裹
- field如果是string类型，需要加引号
## field的类型
int，float， string， bolean
使用如下：
```
 > insert add_test,name=YiHui,phone=110 user_id=21,email="bangzewu@126.com",age=18i,boy=true
```
## 时间戳指定
写入数据不指定时间戳的时候，数据库会使用当前时间自动补齐，如果需要自己指定时间，在最后面添上就行，单位是ns
```
> insert add_test,name=YiHui,phone=110 user_id=22,email="bangzewu@126.com",age=18i,boy=true 1564150279123000000
```
## 指定保存策略插入数据
一个数据库可以有多个保存策略，一个measurement也可以存储不同策略的数据；
不指定保存策略，表示这条数据适用默认的保存策略；如果需要指定保存策略，需要用<font color=red> insert into 保存策略 ... </font>语句来指定
```
insert into "1_d" add_test,name=YiHui2,phone=911 user_id=23,email="bangzewu@126.com",age=18i,boy=true 1564150279123000000
```
## 使用insert修改数据
```
insert into <retention policy> measurement,tagKey=tagValue fieldKey=fieldValue timestamp
```

示例：
```
> select * from add_test where time=1564149327925320596
name: add_test
time                age boy email            name  phone user_id
----                --- --- -----            ----  ----- -------
1564149327925320596         bangzewu@126.com YiHui 110   20
> show tag keys from add_test
name: add_test
tagKey
------
name
phone
> insert add_test,name=YiHui,phone=110 user_id=20,email="bangzewu@126.com",boy=true,age=18i 1564149327925320596
> select * from add_test where time=1564149327925320596
name: add_test
time                age boy  email            name  phone user_id
----                --- ---  -----            ----  ----- -------
1564149327925320596 18  true bangzewu@126.com YiHui 110   20
```
执行insert语句修改已有记录时，有两条必须注意：
- time：指定为要修改记录的时间戳
- tag：所有的tag都必须和要修改的数据一致
如果，这些都满足，会增量修改匹配数据的field，如果不满足，则插入一条新数据

## 删除field？
### field
通过insert可以动态的增加field，但是如果想要删除field，influxdb没有提供这样的操作，但是可以将含有这个field的记录全部删除
### tag
修改记录时根据time+tag values来定位的，如果修改一个tag，对insert而言就是新增了一个point，所以是行不通的，但是可以考虑删除旧数据

# 删除数据
## delete语句
```
DELETE FROM <measurement_name> WHERE [<tag_key>='<tag_value>'] | [<time interval>]
```
从语法中可以看出，只能根据tag或者时间戳来进行匹配

### 删除示例
- 根据时间删除
```
> delete from add_test where time>=1564150279123000000
```
-  根据tag删除
```
> delete from add_test where "name"='YiHui'
```
### 不同保存策略中数据的删除

根据tag进行删除的时候，默认策略和其他策略中的数据都被删除掉了





