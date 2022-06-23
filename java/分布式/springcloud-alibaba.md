# Spring Cloud Alibaba

## Nacos = Eureka+Config+Bus
服务注册与配置中心 ；；；
下载并直接启动，默认用户密码都是nacos
AP和CP都支持，支持切换，默认是AP
切换模式通过PUT访问
```curl -X PUT '$NACOS_SERVER:8848/nacos/v1/ns/operator/switches?entry=serverMode&value=CP'```
**一般来说，如果不需要存储服务级别的信息且服务实例是通过nacos-client注册，并且能够保持心跳上报，那么久可选择AP模式，当前主流的服务比如Spring Cloud 和Dubbo服务，都适用于AP模式，AP模式为了服务的可用性而减弱了一致性，因此AP你试下只支持注册临时实例**
**如果需要在服务级别编辑或者存储配置信息，那么CP是必须的，K8S服务和DNS服务则适用于CP模式。CP模式下支持注册持久化实例，此时是以Raft协议为集群运行模式，该模式下注册实例之前必须先注册服务，如果服务不存在，则返回错误**
### 服务注册发现：
```
		<dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>
            	spring-cloud-starter-alibaba-nacos-discovery
            </artifactId>
        </dependency>
需要在父pom指定版本等信息
<!--spring cloud alibaba 2.1.0.RELEASE-->
      <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-alibaba-dependencies</artifactId>
        <version>2.1.0.RELEASE</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
```

#### 基本配置：
```
server:
  port: 9001

spring:
  application:
    name: nacos-payment-provider
  cloud:
    nacos:
      discovery:
        server-addr: localhost:8848 #配置Nacos地址

management:
  endpoints:
    web:
      exposure:
        include: '*'  #监控

```
### 实现负载均衡：
spring-cloud-starter-alibaba-nacos-discovery集成了ribbon


### 使用配置中心：nacos-config

1. 
```
		<dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
        </dependency>
```
2. 基本配置：
```
spring:
  application:
    name: nacos-config-client
  cloud:
    nacos:
      discovery:
        server-addr: localhost:8848 #Nacos服务注册中心地址
      config:
        server-addr: localhost:8848 #Nacos作为配置中心地址
        file-extension: yaml  #指定yaml格式的配置
        group: DEV_GROUP
        namespace: e69dbe0a-233d-4b73-a173-89e9e8a1d041
```
3. 
dataId的格式：
${prefix}-${spring.profile.active}.${file-extension}
- prefix默认是spring.application.name的值，也可以通过配置项指定：spring.cloud.nacos.config.prefix
- spring.profile.active即为当前环境对应的profile，**当这个值为空的时候，对应的连接符-也将不存在，dataId的拼接格式变为${prefix}.${file-extension}**
- file-extension为配置内容的数据格式，可以通过配置项spring.cloud.nacos.file-extension来配置，目前只支持properties和yaml格式

=========
${spring.application.name}-${spring.profile.active}.${spring.cloud.nacos.config.file.extension}
nacos-config-client-dev.yaml**配置中心的文件名需要是yaml的，需要和file-extension一致**

4. 通过 Spring Cloud 原生注解 @RefreshScope 实现配置自动更新
```
@RestController
@RefreshScope   //SpringCloud原生注解 支持Nacos的动态刷新功能
public class ConfigClientController {

    @Value("${config.info}")
    private String configInfo;

    @GetMapping("/config/info")
    public String getConfigInfo(){
        return configInfo;
    }
}
```
### 命名空间
类似于与java中的package名和类名
最外层是namespace可以用于区分部署环境的，Group和DataId在逻辑上区分两个目标对象

- namespace
	- Group
		- Service
			- Cluster1
				- Instance1
				- Instance2
			- Cluster2 
                - Instance3
				- Instance4
默认情况：
Namespace=public，Group=DEFAULT_GROUP，默认的Cluster是DEFAULT


#### 分组Group
spring.cloud.config.group: groupName
使用group配置指定的组，根据分组获取配置

#### 命名空间namespace
在nacos上创建命名空间，将spacenameId配置到服务，就可以根据命名空间搜索配置文件了
不配置默认public组
spring.cloud.config.namespace: spaceId

### Nacos集群和持久化配置
Nacos自带的内置数据库是derby
目前只支持使用mysql数据库
因为集群部署存在数据一致性的问题，所以使用同一个数据库来完成数据一致

#### 切换数据库为mysql
在conf下拿到MySQL脚本或者从官网获得sql脚本，创建数据库
修改nacos目录下的conf/application.properties文件，增加以下配置
```
spring.datasource.platform=mysql

db.num=1
db.url.0=jdbc:mysql://11.162.196.16:3306/nacos_config?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true
db.user=root
db.password=root
```
#### 集群配置部署
参照官网：```https://nacos.io/zh-cn/docs/cluster-mode-quick-start.html```
1. 在application.properties中配置好数据库信息
2. 配置cluster.conf
	- 配置集群所有节点的IP；IP是hostname -i结果ens后的ip；配置方式是ip:port；配置上所有的节点
3. 启动服务
4. 

## Sentinel：服务降级熔断，流量监控
单独一个组件，支持界面化的细粒度统一配置
下载jar包，直接启动，或者指定port运行 -server.port=xxxx
使用默认用户密码Sentinel登录

- 依赖：
```
	<dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
    </dependency>
```
- 配置：
```
spring:
  application:
    name: cloudalibaba-sentinel-service
  cloud:
    nacos:
      discovery:
        # Nacos服务注册中心地址
        server-addr: localhost:8848
    sentinel:
      transport:
        # sentinel dashboard 地址
        dashboard: localhost:8080
        # 默认为8719，如果被占用会自动+1，直到找到为止
        port: 8719
```
**监控是懒加载的，服务访问一次后，才开始监控**

### 流量控制

直接在sentinel dashboard上可视化设置：

- 资源名：唯一名称，默认请求路径
- 针对来源：Sentinel可以针对调用者进行限流，填写微服务名，默认default
- 阈值类型/单机阈值
	- QPS（每秒的请求数量）：当调用该api的QPS达到阈值的时候，进行限流
	- 线程数：当调用该api的线程数达到阈值的时候，进行限流
- 是否集群：不需要集群
- 流控模式：
	- 直接：api达到限流条件时，直接限流
	- 关联：当关联的资源达到阈值的时候，就限流自己
	- 链路：只记录指定链路上的流量（指定资源从入口资源进来的流量，如果达到阈值，就进行限流【api级别的针对来源】）
- 流控效果：
	- 快速失败：直接失败，抛异常
	- 预热Warm Up：当系统长期处于低水位的情况下，流量突然增加时，直接把系统拉到高水位可能瞬间把系统压垮，通过冷启动的方式，让通过的流量缓慢增加，在一定时间内主键增加到阈值上限，给系统一个预热时间
		- 根据codeFactor（冷加载因子，默认3）的值，从阈值/codeFactor。经过预热时长，才达到设置的QPS阈值
	- 排队等待：匀速排队，让请求以均匀的速度通过，阈值类型必须设置为QPS，否则无效

### 服务降级

- RT：平均响应时间：当1秒内持续进入5个请求，对应时刻的平均响应时间（秒级）均超过阈值，那么在接下来的时间窗口（DegradeRule中的timeWindow，以s为单位）之内，对这个方法的调用都会自动的熔断（抛出DegradeException）。**Sentinel的默认统计RT上限是4900ms，超出都都算作4900**
- 异常比例：当资源的每秒请求量>=5，并且每秒异常总数占通过量的比值超过阈值之后，资源进入降级状态，在接下来的时间窗口之内，对这个方法的调用都会自动的返回，异常比率的阈值范围是[0, 1]，代表0-100%；；；没有达到每秒请求量>=5，有异常会直接报错
- 异常数：当资源近1分钟的异常数超过阈值后进行熔断。由于统计时间窗口是分钟级的，若timeWindow小于60s，则结束熔断后仍可能再进入熔断状态

### 热点限流
使用参数来做限流
@SentinelResource(value="", blockHandler="")
value指定资源名称，blockHandler指定限流条件触发后的处理；如果不配置blockHandler，超过阈值直接返回errorPage
在界面配置的时候指定热点参数下标和阈值

- 限流模式：只能是QPS模式
- 参数索引：方法列表的参数下标，从0开始
- 单机阈值：限流阈值
- 统计窗口时长：秒，与单机阈值一起作用的限流条件
#### 参数例外项
- 参数类型：只支持8中基本类型和String类型
- 参数值：
- 限流阈值：与上面配置的统计窗口时间一同作用
一个热点限流可以设置多个参数例外项

### 系统规则
粗粒度的控制
- LOAD：仅对linux或者linux-like机器生效，系统的load1作为启发指标，进行自适应系统保护，当系统load1超过设置的启发值，且系统当前的并发线程数超过估算的系统容量时才会触发保护（BBR阶段），系统容量由系统的maxQPS * minRT估算得出，设定参考值一般是CPU cores * 2.5
- CPU usage(1.5+版本)：CPU使用率；当系统CPU使用率超过阈值即触发的系统保护（取值0.0-1.0），比较灵敏
- RT：平均响应时间：当单台机器上所有入口流量的平均RT达到阈值即触发系统保护，单位毫秒
- 线程数：当单台机器上所有入口流量的并发线程数达到阈值触发系统保护
- 入口QPS：当单台机器上的所有入口流量的QPS达到阈值即触发系统保护

### 限流设置@SentinelResource
- blockHandlerClass指定限流触发回调函数所在的类
- blockHandler指定限流触发函数
- fallback指定异常回调函数
- exceptionToIgnore：异常忽略：
**指定fallback程序异常直接调用，指定blockHandler限流条件触发调用
如果这两个都使用，达到blockHandler限流条件，调用它的回调函数，不触发fallback**

### sentinel持久化
添加依赖：
```
		<dependency>
            <groupId>com.alibaba.csp</groupId>
            <artifactId>sentinel-datasource-nacos</artifactId>
        </dependency>
```
配置sentinel的数据源信息：
```
spring:
  application:
    name: cloudalibaba-sentinel-service
  cloud:
    nacos:
      discovery:
        # Nacos服务注册中心地址
        server-addr: localhost:8848
    sentinel:
      transport:
        # sentinel dashboard 地址
        dashboard: localhost:8080
        # 默认为8719，如果被占用会自动+1，直到找到为止
        port: 8719
      # 流控规则持久化到nacos
      datasource:
        dsl:
          nacos:
            server-addr: localhost:8848
            data-id: ${spring.application.name}
            group-id: DEFAULT_GROUP
            data-type: json
            rule-type: flow
```
在nacos上配置规则中进行配置

## Seata分布式事务处理

一个全局事务ID（Transaction ID XID） + 3个组件模型
Transaction Coordinator（TC）事务协调器，维护全局事务的运行状态，负责协调并驱动全局事务的提交或者回滚
Transaction Manager（TM）控制全局事务的边界：开始全局事务、提交或者回滚
Resource Manager（RM）控制分支事务，与TC交谈以注册分支事务和报告分支事务的状态，并驱动分支事务提交或者回滚

### 运作流程：



1. TM向TC申请开启一个全局事务，全局事务创建成功并生成一个全局唯一的XID
2. XID在微服务调用链路的上下文传播
3. RM向TC注册分支事务，将其纳入XID对应全局事务的管辖
4. TM向TC发起针对XID的全局提交或者回滚决议
5. TC调度XID下管辖的全部分支事务完成提交或者回滚

@GlobalTransactional

### 粗略执行流程：
- TM开启分布式事务（TM向TC注册全局事务记录）
- 按业务场景，编排数据库、服务等事务内资源（RM向TC汇报资源准备状态）
- TM结束分分布式事务，事务一阶段结束（TM通知TC提交/回滚分布式事务）
- TC汇总事务信息，决定分布式事务是提交还是回滚
- TC通知所有RM提交/回滚资源，事务二阶段结束

### seata模式
- AT：一阶段：业务数据和回滚日志记录在同一个本地事务中提交，释放本地锁和连接资源；二阶段
- TCC：
- SAGA：

### 详细执行流程：
1. 第一个阶段，Seata会拦截业务SQL
	- 解析SQL语义，找到业务SQL需要更新的业务数据，在业务数据被更新前，将其保存为“before image” 
	- 执行业务SQL，更新业务数据，在业务数据更新之后，
	- 将其保存为“after image”，最后生成行锁
以上操作全部在一个数据库事务内完成，这样保证了一阶段操作的原子性
2. 二阶段
	- 二阶段回滚：二阶段如果是回滚的话，需要混滚一阶段已经执行的SQL，回滚方式使用before image还原业务数据，还原之前，需要校验脏写，对比数据库当前业务数据和after image，如果一直，就进行还原，如果不一致，说明有脏写，需要转人工处理
	- 二阶段提交：将一阶段保存的快照数据和行锁删除