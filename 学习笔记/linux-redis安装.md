###  地址
https://redis.io/download
### 安装
- 下载：wget https://download.redis.io/releases/redis-6.2.6.tar.gz
- 解压：$ tar xzf redis-6.2.6.tar.gz
可以将文件移动到自己的目录

- 切换到目录：$ cd redis-6.2.6
- 编译：$ make


- 安装：make PREFIX=/usr/local/redis install
`PREFIX=` 这个关键字的作用是编译的时候用于指定程序存放的路径。比如现在就是指定了redis必须存放在/usr/local/redis目录。假设不添加该关键字Linux会将可执行文件存放在/usr/local/bin目录， 库文件会存放在/usr/local/lib目录。配置文件会存放在/usr/local/etc目录。其他的资源文件会存放在usr/local/share目录。这里指定目录也方便后续的卸载，后续直接rm -rf /usr/local/redis 即可删除redis
### 启停
- 后台启动：./bin/redis-server& ./redis.conf

----- 当前目录下的所有文件与子目录权限 chmod -R u+x *

使用redis-cli连接redis
- 查看配置`config get *`
- 查看进程`ps -ef | grep redis`

./bin/redis-cli shutdown

**
redis.conf默认的bind值是127.0.0.1，即本机，因此导致Java客户端不能连接
daemonize yes 以守护进程方式启动
protected-mode no 关闭保护模式
requirepass 设置密码
**

systemctl status firewalld.service
systemctl stop firewalld.service