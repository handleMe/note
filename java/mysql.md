## Mysql索引
B+树
### 一般索引
– 直接创建索引
CREATE INDEX index_name ON table(column(length))
– 修改表结构的方式添加索引
ALTER TABLE table_name ADD INDEX index_name ON (column(length))
– 创建表的时候同时创建索引
```
CREATE TABLE `table` (
`id` int(11) NOT NULL AUTO_INCREMENT ,
`title` char(255) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL ,
`content` text CHARACTER SET utf8 COLLATE utf8_general_ci NULL ,
`time` int(10) NULL DEFAULT NULL ,
PRIMARY KEY (`id`),
INDEX index_name (title(length))
)
```
– 删除索引
DROP INDEX index_name ON table

### 唯一索引
与普通索引类似，不同的就是：索引列的值必须唯一，但允许有空值（注意和主键不同）。如果是组合索引，则列值的组合必须唯一，创建方法和普通索引类似
```
–创建唯一索引
CREATE UNIQUE INDEX indexName ON table(column(length))
–修改表结构
ALTER TABLE table_name ADD UNIQUE indexName ON (column(length))
–创建表的时候直接指定
CREATE TABLE `table` (
`id` int(11) NOT NULL AUTO_INCREMENT ,
`title` char(255) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL ,
`content` text CHARACTER SET utf8 COLLATE utf8_general_ci NULL ,
`time` int(10) NULL DEFAULT NULL ,
PRIMARY KEY (`id`),
UNIQUE indexName (title(length))
```
### 单列索引、多列索引
多个单列索引与单个多列索引的查询效果不同，因为执行查询时，MySQL只能使用一个索引，会从多个索引中选择一个限制最为严格的索引

### 组合索引

例如上表中针对title和time建立一个组合索引：ALTER TABLE article ADD INDEX index_titme_time (title(50),time(10))。建立这样的组合索引，其实是相当于分别建立了下面两组组合索引：
–title,time
–title
为什么没有time这样的组合索引呢？这是因为MySQL组合索引“最左前缀”的结果

### 聚集索引，非聚集索引
聚集索引：每张表一个；就是索引和数据存储在一个文件

非聚集索引：存储在索引文件中

Innodb
1.主键
2.没有，则依据第一个不为空的唯一键索引
3.都没有，生成一个隐式的聚簇索引
其他索引都是非聚集的索引

### 索引失效原因
- 单列索引无法存储null值，复合索引无法存储全是null的值，存储了null值就会放弃使用索引
- 非最左匹配，用like的时候，前面使用了通配符
- 使用了or连接，如果要使or连接的条件也使用索引，就要将or条件中的所有列都加上索引
- 多列索引，也适用最左匹配，
- 计算、函数、类型转换（自动或者手动）会失效，比如字符串没有加引号
- 使用不等于 !=和<> 
- 判断是否是null ； is null 、is not null

### union和union all区别
union去重，默认排序
union all直接将所有的结果保留，效率高