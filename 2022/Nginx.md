# Nginx

为什么选择Nginx
- 更快
- 高扩展性
- 高可靠性
- 低内存消耗
- 单机支持大量并发连接 10W+
- 热部署

## 前言

### 修改linux配置
/etc/sysctl.conf
```
fs.file-max = 999999              指一个进程可以打开的最大句柄数，限制最大连接数
net.ipv4.top_tw_reuse = 1         1表示允许将TIME-WAIT的socket重新用于新的TCP连接
net.ipv4.top_keepalive_time = 600 表示keepalive启用时，TCP发送keepaliv消息的频度，默认是两小时，配置小可以更快的清理无效的连接
net.ipv4.top_fin_timeout = 30     表示服务器主动关闭连接，socket保持在FIN-WAIT-2状态的最大时间
net.ipv4.top_max_tw_buckets = 5000 表示操作系统允许TIME_WAIT套接字数量的最大值，超过后，此状态的套接字会被like清除，并打印警告信息，默认180000，过多会使服务器变慢
net.ipv4.ip_local_port_range = 1024   61000 这个参数定义了在UDP和TCP连接中本地（不包括连接的远端）端口的取值范围
net.core.netdev_max_backlog = 8096  当网卡接收数据包的速度大于内核处理的速度时，会有一个队列保存这些数据包。这个参数表示该队列的最大值
net.core.rmem_default = 262144      这个参数表示内核套接字接收缓存区默认的大小
net.core.wmem_default = 262144      这个参数表示内核套接字发送缓存区默认的大小
net.core.rmem_max = 2097152         这个参数表示内核套接字接收缓存区的最大大小
net.core.wmem_max = 2097152         这个参数表示内核套接字发送缓存区的最大大小
net.ipv4.top_syncookies = 1  该参数与性能无关，用于解决TCP的SYN攻击
net.ipv4.top_max_syn_backlog = 1024  表示TCP三次握手建立阶段接受SYN请求队列的最大长度，默认1024，设置大一些可以使Nginx繁忙时来不及accept的新连接，不至于丢失
```
执行sysctl -p使配置生效

下载Nginx后，
./configure  检测操作系统内二和已安装软件，参数解析，中检目录生成以及根据参数生成一些源码文件、Makefile等
make  编译源码，生成目标文件、最终的二进制文件
make install 根据参数将Nginx部署到指定的安装目录，包括相关目录建立和二进制文件、配置文件的复制


### 命令执行和控制
	默认情况下，Nginx被安装在/usr/local/nginx中，二进制文件路径为：/usr/local/nginx/sbin/nginx；配置文件：/usr/lcoal/conf/nginx.conf；可以指定其他的安装路径
- 默认启动方式：直接执行二进制程序：/usr/local/nginx/sbin/nginx
这样会读取默认路径下的配置文件
- 指定配置文件启动：/usr/local/nginx/sbin/nginx/ -c /tmp/nginx.conf
- 指定安装目录启动：/usr/local/nginx/sbin/nginx -p /usr/local/nginx/
- 指定全局配置项的启动方式：/usr/local/nginx/sbin/nginx -g "pid /var/nginx/test.pid;"
- 查看版本号：/usr/local/nginx/sbin/nginx -v
- 强制停止服务：/usr/local/nginx/sbin/nginx -s stop
- 优雅的停止服务：/usr/local/nginx/sbin/nginx -s quit
- 使运行中的服务重新加载配置文件：/usr/local/nginx/sbin/nginx -s reload
- 日志文件回滚：/usr/local/nginx/sbin/nginx -s reopen 可以先将日志文件备份，然后使用命令重新打开日志文件，装使得日志文件不至于过大

## Nginx的配置
	一般使用一个master进程和多个worker进程，worker进程的数量一般和服务器上CPU核心数相同；这时候，master线程时负责监控和管理worker进程的，master进程则是真正提供服务的
	Nginx也支持单进程（master进程）
**使用多个进程的好处**
- 这样worker进程只关注与管理worker进程，当worker进程出现错误coredump的时候，master进程会l立刻重新启动新的worker继续服务
- 健壮性，一个worker出现错误，其他进程还是可以提供服务

### 块配置项
由一个配置项名称和一堆大括号组成；块配置项可以继承，内层继承外层
### 注释
	#
### 单位
指定空间大小时，不用都指定到字节，可以使用K代表千字节，M代表兆字节
指定时间的时候，可以使用ms毫秒、s秒、m分钟，h小时，d天，w周，M月，y年

### 变量
某些模块支持参数，使用$表示；

### 用于调试和定位问题的配置项

#### 是否以守护进程的方式运行nginx
daemon on|off 默认是on 这样不会在任何终端上打印信息，也不会被终端所产生的的信息打断进程
提供关闭的模式，视为了可进行调试

#### 是否以master/worker方式工作
master_process on|off 默认on

### error日志的设置
error_log/path/file level;
默认：error_log logs/error.log error;
日志文件可以使一个具体文件，也可以是/dev/null 这样就不输出日志了
日志级别：debug、info、notice、warn、eror、crit、alert、emerg；依次递增；
设定一个级别后，高于该级别的日志都会被输出
如果设置级别是debug，必须在configure加入 --with-debug配置项。

### debug_points [stop|abort]

### debug_connection [IP|CIDR]
属于事件类配置，必须放在events{...}中才有效，值可以使IP或者CIDR地址
需要在configure时已经加入了--with-debug参数，否则不会生效
```
events {
	debug_connection 10.244.11.14;
	debug_connection 10.244.24.0/24;
}
```
**这样只有配置的IP地址请求才会输出debug级别的日志，其他请求仍沿用error_log中配置的级别**
### 限制coredump核心转出文件的大小

worker_rlimit_core size;
在Linux系统中，当进程发生错误或收到信号终止后，系统会将进程执行的内存内容写入一个文件（core文件），用作调试，这就是核心转储（core dumps）。当Nginx进程出现一些非法操作导致进程被系统强制结束后，会生成核心转储core文件；文件中有当时的堆栈、寄存器想你想，从而帮助我们定位问题；这些信息不都是用户所需要的，如果不限制，可能会达到几GB，通过限制文件大小，规避文件过大引发的问题
### 指定coredump文件生成目录
working_directory path

### 环境变量
env VAR|VAR=VALUE
env TESTPAT/tmp/;

### 引入其他配置文件
include /path/file;
可以将其他文件嵌入当前的nginx.conf中
`include mime.types;`
`include vhost/*.conf;`

### pid文件路径
pid path/file
默认：pid logs/nginx.pid;
默认与configure执行时的参数 --pid-path所指定的路径时相同的，可以随时修改，但应该保证nginx有权限在对应的位置创建pid文件，pid文件直接影响nginx是否可以运行
### worker进程运行的用户及用户组
user username[groupname];
默认：user nobody nobody;
user用于设置master进程启动后，fork出的worker进程运行在那个用户和用户组下，按照user username；设置时，用户组与用户名相同
当用户在configure命令执行的视乎使用了参数--user=username和--group=groupname，此时nginx.conf将使用参数中指定的用户和用户组
### worker进程可以打开的最大文件数
worker_rlimit_nofile limit;
### 限制信号队列
worker_rlimit_sigpending limit;
设置每个用户发往信号队列的大小

## 优化性能的配置项

### worker进程的个数
worker_processes number;
默认：1
在master/worker运行方式下，定义worker进程的个数。
每个worker进程都是单线程进程；如果阻塞式调用比较多（比如大量的静态文件读取），可以配置与CPU内核相同的进程数；如果阻塞式调用多，就需要配置多一些worker进程
	一般情况下都配置等同于COU内核数的进程数，使用worker_cpu_affinity配置来绑定CPU内核
### 绑定CPU内核和worker进程
worker_cpu_affinity cpumask[cpumask...]

## 事件类配置项
### 是否打开accept锁
accept_mutex [on|off];
默认on
负载均衡锁，关闭可以使TCP连接耗时更短，但是worker的负载会不均衡；
### lock文件的路径
lock_file path/file;
默认：lock_file logs/nginx.lock;
如果打开了accept锁，并且由于编译程序、操作系统架构等因素导致nginx不支持原子锁，才会用到这个文件来实现accept锁

### 使用accept锁到真正建立连接之间的延迟时间
accept_mutex_delay Nms；
默认：500ms
同一时间只有一个worker进程能够获取accept锁，这个accept锁不是阻塞锁；获取不到会like返回；如果一个进程获取accept锁失败，至少要等延时时间间隔后才能再次试图获取锁；
### 批量建立新连接
multi_accept [on|off];
默认off
时间模型通知有新连接时，尽可能对本次调度中客户端发起的所有TCP请求都建立连接
### 选择时间模型
user[kqueue|rtsig|epoll|/dev/poll|select|poll|eventport];
默认：Nginx会使用最合适的事件模型
对于linux系统来说，可供选择的有poll、select、epoll三种，epoll性能最高