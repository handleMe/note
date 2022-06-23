[TOC]

#  创建数据库

CREATE DATABASE "db_name"
# 显示所有数据库
SHOW DATABASES
# 删除数据库
DROP DATABASE "db_name"

# 使用数据库
USE mydb

# 显示该数据库中的表
SHOW MEASUREMENTS

# 创建表
# 直接在插入数据的时候指定表名（weather就是表名）
insert weather,altitude=1000,area=北 temperature=11,humidity=-4

# 删除表
DROP MEASUREMENT "measurementName"

##InfluxDB没有提供直接删除Points的方法，但是它提供了Retention Policies。
##主要用于指定数据的保留时间：当数据超过了指定的时间之后，就会被删除。

#查看当前数据库的Retention Policies
SHOW RETENTION POLICIES ON "testDB"

#创建新的Retention Policies
CREATE RETENTION POLICY "rp_name" ON "db_name" DURATION 30d REPLICATION 1 DEFAULT
其中：
rp_name：策略名
db_name：具体的数据库名
30d：保存30天，30天之前的数据将被删除
它具有各种时间参数，比如：h（小时），w（星期）
REPLICATION 1：副本个数，这里填1就可以了
DEFAULT 设为默认的策略

#修改Retention Policies
ALTER RETENTION POLICY "rp_name" ON db_name" DURATION 3w DEFAULT

#删除Retention Policies
DROP RETENTION POLICY "rp_name" ON "db_name"

#####连续查询 Continuous Queries
# 查询当前的连续查询
SHOW CONTINUOUS QUERIES

# 创建新的Continuous Queries
CREATE CONTINUOUS QUERY cq_30m ON testDB BEGIN SELECT mean(temperature) INTO weather30m FROM weather GROUP BY time(30m) END

· cq_30m：连续查询的名字
· testDB：具体的数据库名
· mean(temperature): 算平均温度
· weather： 当前表名
· weather30m： 存新数据的表名
· 30m：时间间隔为30分钟

-- 当我们插入新数据之后，可以发现数据库中多了一张名为weather30m(里面已经存着计算好的数据了)。
-- 这一切都是通过Continuous Queries自动完成的。

# 删除Continuous Queries
DROP CONTINUOUS QUERY <cq_name> ON <database_name>

### 用户管理
-- 以下语句都可以直接在InfluxDB的Web管理界面中调用
# 显示用户
SHOW USERS
# 创建用户
CREATE USER "username" WITH PASSWORD 'password'
# 创建管理员权限的用户
CREATE USER "username" WITH PASSWORD 'password' WITH ALL PRIVILEGES

# 删除用户
DROP USER "username"