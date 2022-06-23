# springcloud

## Eureka服务注册与发现

### 服务注册中心：
配置

```
server:
  port: 7001

eureka:
  instance:
    hostname: eureka7001.com #eureka服务端的实例名称
  client:
    #false表示不向注册中心注册自己
    register-with-eureka: false
    #false表示自己端就是注册中心，我的职责就是维护服务实例，并不需要去检索服务
    fetch-registry: false
    service-url:
      #设置与Eureka Server交互的地址查询服务和注册服务都需要依赖这个地址
      #defaultZone: http://eureka7002.com:7002/eureka/  #互相注册；多个使用逗号分隔
      defaultZone: http://eureka7001.com:7001/eureka/ #单机模式
  #server:
    #关闭自我保护机制，保证不可用服务被及时剔除
    #enable-self-preservation: false
    #eviction-interval-timer-in-ms: 2000
```
代码
```
@SpringBootApplication
@EnableEurekaServer
public class EurekaMain7001 {
    public static void main(String[] args) {
        SpringApplication.run(EurekaMain7001.class,args);
    }
}
```
### 微服务注册

```
eureka:
  client:
    #表示是否将自己注册进EurekaServer默认为true
    register-with-eureka: true
    #是否从EurekaServer抓取已有的注册信息，默认为true。单节点无所谓，集群必须设置为true才能配合ribbon使用负载均衡
    fetch-registry: true
    service-url:
      defaultZone: http://localhost:7001/eureka #单机版
      #defaultZone: http://eureka7001.com:7001/eureka,http://eureka7002.com:7002/eureka  #集群版
  #actuator微服务信息完善
  instance:
    instance-id: payment8001
    prefer-ip-address: true #访问路径可以显示IP地址
    #Eureka客户端向服务端发送心跳的时间间隔，单位为秒（默认是30秒）
    #lease-renewal-interval-in-seconds: 1
    #Eurekaf服务端在收到最后一次心跳后等待时间上限，单位为秒（默认是90秒），超时将剔除服务
    #lease-expiration-duration-in-seconds: 2
```
代码

```
@SpringBootApplication
@EnableEurekaClient
//@RibbonClient(name = "CLOUD-PROVIDER-SERVICE",configuration = MySelfRule.class)
public class OrderMain80 {
    public static void main(String[] args) {
        SpringApplication.run(OrderMain80.class,args);
    }
}
```
### eureka自我保护

Eureka Server 在运行期间会去统计心跳失败比例在 15 分钟之内是否低于 85%，如果低于 85%，Eureka Server 会将这些实例保护起来，让这些实例不会过期，但是在保护期内如果服务刚好这个服务提供者非正常下线了，此时服务消费者就会拿到一个无效的服务实例，此时会调用失败，对于这个问题需要服务消费者端要有一些容错机制，如重试，断路器等
可以通过参数配置来关闭
```
	#server:
    #关闭自我保护机制，保证不可用服务被及时剔除
    #enable-self-preservation: false
    #eviction-interval-timer-in-ms: 2000
```

## Zookeeper实现服务注册

```
		<dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-zookeeper-config</artifactId>
        </dependency>
配置文件：
server:
  port: 80
spring:
  application:
    name: cloud-consumer-order
  cloud:
    #注册到zookeeper地址
    zookeeper:
      connect-string: 192.168.88.131:2181

启动类：
@SpringBootApplication
@EnableDiscoveryClient
public class ZookApp {
   public static void main(String[] args){
       SpringApplication.run(ZookApp.class, args);
       System.out.println("hello springcloud");
   }
}
```

## Consul服务注册

是什么
	https://www.consul.io/intro/index.html
能干嘛
	服务发现
		提供HTTP/DNS两种发现方式
	健康检测
		支持多种方式,HTTP、TCP、Docker、shell脚本定制化
	KV存储
		Key、Value的存储方式
	多数据中心
		Consul支持多数据中心
	可视化界面
下哪下
	https://www.consul.io/downloads.html
怎么玩
	https://www.springcloud.cc/spring-cloud-consul.html

```
spring:
  application:
    name: cloud-consumer-order
###consul服务注册中心
  cloud:
    consul:
      host: localhost
      port: 8500
      discovery:
        #hostname: 127.0.0.1
        service-name: ${spring.application.name}
```
```
spring:
  application:
    name: consul-provider-payment
###consul注册中心地址
  cloud:
    consul:
      host: localhost
      port: 8500
      discovery:
        #hostname: 127.0.0.1
        service-name: ${spring.application.name}
```

## Ribbon负载均衡

eureka中带有ribbon依赖
### 负载策略：
com.netflix.loadbalancer.RoundRobinRule**轮询**
com.netflix.loadbalancer.RandomRule**随机**
com.netflix.loadbalancer.RetryRule**轮询-重试**：先按照RoundRobinRule的策略获取服务,如果获取服务失败则在指定时间内进行重试,获取可用的服务
WeightedResponseTimeRule **加权**：对RoundRobinRule的扩展,响应速度越快的实例选择权重越多大,越容易被选择
BestAvailableRule**基于并发量进行选择**：会先过滤掉由于多次访问故障而处于断路器跳闸状态的服务,然后选择一个并发量最小的服务
AvailabilityFilteringRule**基于并发量进行选择**：先过滤掉故障实例,再选择并发较小的实例
ZoneAvoidanceRule**默认规则**：默认规则,复合判断server所在区域的性能和server的可用性选择服务器

#### 特殊化定制负载均衡策略
```
自定义策略的注入需要在@ComponentScan扫描不到的地方，否则将会作为全局的策略被所有ribbon客户端共享，而不是单独定制的策略使用；
@Configuration
public class MySelfRule {
    @Bean
    public IRule myRule(){
        return new RandomRule();
    }
}
```
```
//原子类
    private AtomicInteger atomicInteger = new AtomicInteger(0);

    public final int getAndIncrement(){
        int current;
        int next;
        do{
            current = this.atomicInteger.get();
            //超过最大值，为0，重新计数 2147483647 Integer.MAX_VALUE，不直接写方法是因为方法运行时每次都会创建一个栈区，而写死就不会，涉及Jvm调优
            next = current >= 2147483647 ? 0 : current + 1;
            //自旋锁
        }while (!this.atomicInteger.compareAndSet(current,next));
        System.out.println("*****第几次访问，次数next："+next);
        return next;
    }

    @Override
    public ServiceInstance instances(List<ServiceInstance> serviceInstance) {
        int index = getAndIncrement() % serviceInstance.size();
        return serviceInstance.get(index);
    }
```


## OpenFeign远程调用

```
配置
#客户端默认的等待时间只有1秒，业务处理可能会超过1秒，下面是相关时间的配置
#设置feign 客户端超时时间（openFeign默认支持ribbon）
ribbon:
#指的是建立连接后从服务器读取到可用资源所用的时间
  ReadTimeout: 5000
#指的是建立连接所用的时间，适用于网络状况正常的情况下，两端连接所用的时间
  ConnectTimeout: 5000

```

```
@SpringBootApplication
@EnableFeignClients //开启Feign
public class OrderFeignMain80 {

    public static void main(String[] args) {
        SpringApplication.run(OrderFeignMain80.class,args);
    }
}
```

```
@Component
@FeignClient(value = "CLOUD-PROVIDER-SERVICE")  //指定调用哪个微服务
public interface PaymentFeignService {

    @GetMapping(value = "/payment/get/{id}")    //哪个地址
    CommonResult<Payment> getPaymentById(@PathVariable("id") Long id);

    @GetMapping(value = "/payment/feign/timeout")
    public String paymentFeignTimeout();

}
```
```
调用日志的打印级别
@Configuration
public class FeignConfig {
    @Bean
    Logger.Level feignLoggerLevel(){
        return Logger.Level.FULL;
    }
}
```

## Hystrix熔断器

**服务雪崩**
通常一个模块下的某个实例失败后，这个模块还在接收流量，然后这个有问题的模块还调用了其他模块，这样就会发生级联故障，或者叫服务雪崩

服务降级
	服务器忙,请稍后再试,不让客户端等待并立刻返回一个友好提示,fallback
	哪些情况会发出降级
		程序运行异常
		超时
		服务熔断触发服务降级
		线程池/信号量也会导致服务降级
服务熔断
	类比保险丝达到最大服务访问后,直接拒绝访问,拉闸限电,然后调用服务降级的方法并返回友好提示
	就是保险丝
		服务的降级->进而熔断->恢复调用链路
服务限流
	秒杀高并发等操作,严禁一窝蜂的过来拥挤,大家排队,一秒钟N个,有序进行

### 服务降级

一旦调用服务方法失败并抛出了错误信息后,会自动调用@HystrixCommand标注好的fallbckMethod调用类中的指定方法

为单个方法定制降级策略
```
主启动类激活
	@EnableCircuitBreaker

降级方法上注解
@HystrixCommand(fallbackMethod = "paymentTimeOutFallBackMethod",commandProperties = {
            @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds",value = "1500")
    })
public String paymentInfo_TimeOut(@PathVariable("id") Integer id){
        int age = 10/0;
        return paymentHystrixService.paymentInfo_TimeOut(id);
    }

    public String paymentTimeOutFallBackMethod(@PathVariable("id") Integer id){
        return "我是消费者80，对方支付系统繁忙，请稍后再试，o(╥﹏╥)o";
    }
```

为一个类中的方法指定默认的降级方法
```
@DefaultProperties(defaultFallback = "payment_Global_FallbackMethod")
public class OrderHystrixController {
    @Resource
    private PaymentHystrixService paymentHystrixService;

    @GetMapping("/consumer/payment/hystrix/ok/{id}")
    public String paymentInfo_OK(@PathVariable("id") Integer id){
        return paymentHystrixService.paymentInfo_OK(id);
    }

    @GetMapping("/consumer/payment/hystrix/timeout/{id}")
    @HystrixCommand
    public String paymentInfo_TimeOut(@PathVariable("id") Integer id){
        int age = 10/0;
        return paymentHystrixService.paymentInfo_TimeOut(id);
    }
    /**
     * 全局 fallback 方法
     * @return
     */
    public String payment_Global_FallbackMethod(){
        return "Global异常处理信息，请稍后再试。/(╥﹏╥)/~~";
    }
}
```
全局降级方法指定（直接指定对某个服务调用的降级方法）
```
@Component
@FeignClient(value = "CLOUD-PROVIDER-HYSTRIX-PAYMENT" ,fallback = PaymentFallbackService.class)
public interface PaymentHystrixService {

    @GetMapping("/payment/hystrix/ok/{id}")
    public String paymentInfo_OK(@PathVariable("id") Integer id);

    @GetMapping("/payment/hystrix/timeout/{id}")
    public String paymentInfo_TimeOut(@PathVariable("id") Integer id);
}

@Component
public class PaymentFallbackService implements PaymentHystrixService {
    @Override
    public String paymentInfo_OK(Integer id) {
        return "----PaymentFallbackService fall back-paymentInfo_OK,o(╥﹏╥)o";
    }

    @Override
    public String paymentInfo_TimeOut(Integer id) {
        return "----PaymentFallbackService fall back-paymentInfo_TimeOut,o(╥﹏╥)o";
    }
}
```
### 服务熔断

```
//服务熔断
    @HystrixCommand(fallbackMethod = "paymentCircuitBreaker_fallback",commandProperties = {
            @HystrixProperty(name = "circuitBreaker.enabled",value = "true"),   //是否开启断路器
            @HystrixProperty(name = "circuitBreaker.requestVolumeThreshold",value = "10"),  //请求次数
            @HystrixProperty(name = "circuitBreaker.sleepWindowInMilliseconds",value = "10000"),    //时间窗口期
            @HystrixProperty(name = "circuitBreaker.errorThresholdPercentage",value = "60"),    //失败率达到多少后跳闸
    })
    public String paymentCircuitBreaker(@PathVariable("id") Integer id){
        if(id < 0){
            throw new RuntimeException("******id 不能为负数");
        }
        String serialNumber = IdUtil.simpleUUID();  //UUID.randomUUID();

        return Thread.currentThread().getName()+"\t"+"调用成功，流水号："+serialNumber;
    }
    public String paymentCircuitBreaker_fallback(@PathVariable("id") Integer id){
        return "id 不能负数，请稍后再试，o(╥﹏╥)o  id："+id;
    }
```

1. 当满足一定的阈值的时候(默认10秒钟超过20个请求次数)
2. 当失败率达到一定的时候(默认10秒内超过50%的请求次数)
3. 到达以上阈值,断路器将会开启
4. 当开启的时候,所有请求都不会进行转发
5. 一段时间之后(默认5秒),这个时候断路器是半开状态,会让其他一个请求进行转发. 如果成功,断路器会关闭,若失败,继续开启.重复4和5
**断路器半开：服务熔断后，一段时间之后，半路齐会允许一个请求，入股哦成功，断路器关闭，如果失败，继续开启**
### 服务限流

```
@HystrixCommand(fallbackMethod = "payment_fallback",
	groupKey="orderGroup",
	threadPoolKey="getPersonById",
	threadPoolProperties={
		@HystrixProperty(name="coreSize", value="2"),
		@HystrixProperty(name="maxQueueSize", value="3"),
		@HystrixProperty(name="queueSizeRejectionThreshold", value="1"),
	}
)
public Result get(){
}
```
CommandGroupKey：配置全局唯一标识服务分组的名称，比如，库存系统就是一个服务分组。当我们监控时，相同分组的服务会聚合在一起，必填选项。
CommandKey：配置全局唯一标识服务的名称，比如，库存系统有一个获取库存服务，那么就可以为这个服务起一个名字来唯一识别该服务，如果不配置，则默认是简单类名。
ThreadPoolKey：配置全局唯一标识线程池的名称，相同线程池名称的线程池是同一个，如果不配置，则默认是分组名，此名字也是线程池中线程名字的前缀。
ThreadPoolProperties：配置线程池参数，coreSize配置核心线程池大小和线程池最大大小，keepAliveTimeMinutes是线程池中空闲线程生存时间（如果不进行动态配置，那么是没有任何作用的），maxQueueSize配置线程池队列最大大小，
queueSizeRejectionThreshold限定当前队列大小，即实际队列大小由这个参数决定，通过改变queueSizeRejectionThreshold可以实现动态队列大小调整。
CommandProperties：配置该命令的一些参数，如executionIsolationStrategy配置执行隔离策略，默认是使用线程隔离，此处我们配置为THREAD，即线程池隔离。

此处可以粗粒度实现隔离，也可以细粒度实现隔离，如下所示。

服务分组+线程池：粗粒度实现，一个服务分组/系统配置一个隔离线程池即可，不配置线程池名称或者相同分组的线程池名称配置为一样。
服务分组+服务+线程池：细粒度实现，一个服务分组中的每一个服务配置一个隔离线程池，为不同的命令实现配置不同的线程池名称即可。
混合实现：一个服务分组配置一个隔离线程池，然后对重要服务单独设置隔离线程池。本demo可以用Jmeter模拟多线程调用来验证结果
那么，这个是Hystrix基于线程池+队列的方式实现的限流，当然，还有另外一种，基于信号量来实现的。那么什么是信号量把，说白了就是计数器。当 计数器 + 1  > 设置的最大并发数 时，就限流，或者走降级渠道

### 服务监控hystrixDashboard
需要依赖：
```
		<dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
        </dependency>
```
```
@SpringBootApplication
@EnableHystrixDashboard
public class HystrixDashboardMain9001 {
    public static void main(String[] args) {
        SpringApplication.run(HystrixDashboardMain9001.class,args);
    }


}
```
## Gateway网关

Route(路由)
	路由是构建网关的基本模块,它由ID,目标URI,一系列的断言和过滤器组成,如断言为true则匹配该路由
Predicate(断言)
	参考的是Java8的java.util.function.Predicate
开发人员可以匹配HTTP请求中的所有内容(例如请求头或请求参数),如果请求与断言相匹配则进行路由
Filter(过滤)
	指的是Spring框架中GatewayFilter的实例,使用过滤器,可以在请求被路由前或者之后对请求进行修改.


```
server:
  port: 9527

spring:
  application:
    name: cloud-gateway
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true   #开启从注册中心动态创建路由的功能，利用微服务名进行路由
      routes:
        - id: payment_routh #payment_routh    #路由的ID，没有固定规则但要求唯一，简易配合服务名
          #uri: http://localhost:8001         #匹配后提供服务的路由地址
          uri: lb://cloud-provider-service   #匹配后提供服务的路由地址
          predicates:
            - Path=/payment/get/**          #断言，路径相匹配的进行路由

        - id: payment_routh2 #payment_routh   #路由的ID，没有固定规则但要求唯一，简易配合服务名
          #uri: http://localhost:8001          #匹配后提供服务的路由地址
          uri: lb://cloud-provider-service     #匹配后提供服务的路由地址
          predicates:
            - Path=/payment/lb/**             #断言，路径相匹配的进行路由
            #- After=2020-03-15T15:35:07.412+08:00[GMT+08:00]
            #- Cookie=username,zzyy
            #- Header=X-Request-Id, \d+ #请求头要有X-Request-Id属性并且值为整数的正则表达式
            #- Host=**.atguigu.com
            #- Method=GET
            #- Query=username, \d+ #要有参数名username并且值还要啥整数才能路由
# 注册服务
eureka:
  instance:
    hostname: cloud-gateway-service
  client:
    register-with-eureka: true
    fetch-registry: true
    service-url:
      defaultZone: http://eureka7001.com:7001/eureka
```
根据指定的Predicates断言规则可以将访问转发至对应的uri
可以使用时间规则进行无感的服务升级
可以做简单的权限验证
可以根据访问ip或者参数让用户访问不同的服务（就近匹配服务器或者VIP等）

## springcloud config
集中管理配置文件
不同环境不同配置，动态化的配置更新，分环境比如dev/test/prod/beta/release
运行期间动态调整配置，不再需要在每个服务部署的机器上编写配置文件，服务会向配置中心同意拉去配置自己的信息
当配置发生改变时，服务不需要重启即可感知到配置的变化并应用新的配置
将配置信息以REST接口的形式暴露
	post/crul访问刷新即可...

由于SpringCloud Config默认使用GIt来存储配置文件(也有其他方式，比如支持SVN和本地文件)，但最推荐还是Git，而且使用的是http/https访问的形式
### 服务端构建
```
@SpringBootApplication
@EnableConfigServer
public class ConfigCenterMain3344 {
    public static void main(String[] args) {
        SpringApplication.run(ConfigCenterMain3344.class,args);
    }
}

spring:
  application:
    name: cloud-config-center
  cloud:
    config:
      server:
        git:
          uri: git@github.com:TNoOne/cloud-config.git #github仓库上面的git仓库名字
          ##搜索目录
          search-paths:
            - cloud-config
      #读取分支
      label: master
eureka:
  client:
    service-url:
      defaultZone: http://localhost:7001/eureka #注册进eureka
```

### 客户端配置
这里使用bootstrap.yml来作为服务自身的配置，使用application.yml来作为配置中心的公共配置
**bootstrap.yml（bootstrap.properties）与application.yml（application.properties）执行顺序**
bootstrap.yml（bootstrap.properties）用来程序引导时执行，应用于更加早期配置信息读取，如可以使用来配置application.yml中使用到参数等
application.yml（application.properties) 应用程序特有配置信息，可以用来配置后续各个模块中需使用的公共参数等。
加载顺序
**bootstrap.yml > application.yml > application-dev(prod).yml**
```
spring:
  application:
    name: config-client
  cloud:
    #Config客户端配置
    config:
      label: master #分支名称
      name: config #配置文件名称
      profile: dev #读取后缀名称 上述3个综合：master分支上config-dev.yml的配置文件被读取 http://config-3344.com:3344/master/config-dev.yml
      uri: http://localhost:3344 #配置中心地址
#服务注册到eureka地址
eureka:
  client:
    service-url:
      defaultZone: http://localhost:7001/eureka
```
### 服务端配置

```
spring:
  application:
    name: config-client
  cloud:
    #Config客户端配置
    config:
      label: master #分支名称
      name: config #配置文件名称
      profile: dev #读取后缀名称 上述3个综合：master分支上config-dev.yml的配置文件被读取 http://config-3344.com:3344/master/config-dev.yml
      uri: http://localhost:3344 #配置中心地址
#服务注册到eureka地址
eureka:
  client:
    service-url:
      defaultZone: http://localhost:7001/eureka
```

### 刷新配置
curl -X POST "http://localhost:3355/actuator/refresh"
手动访问配置配置中心的刷新连接
## 消息总线BUS
由于刷新config配置需要逐个调用配置连接，太过繁琐；引入消息总线，降低维护难度

**Bus支持两种消息代理:RabbitMQ和Kafka**
### 服务端配置
```
spring:
  application:
    name: config-client
  cloud:
    #Config客户端配置
    config:
      label: master #分支名称
      name: config #配置文件名称
      profile: dev #读取后缀名称 上述3个综合：master分支上config-dev.yml的配置文件被读取 http://config-3344.com:3344/master/config-dev.yml
      uri: http://localhost:3344 #配置中心地址
  #rabbit相关配置 15672是web管理界面的端口，5672是MQ访问的端口
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest
#服务注册到eureka地址
eureka:
  client:
    service-url:
      defaultZone: http://localhost:7001/eureka

#暴露bus刷新配置的端点
management:
  endpoints:  #暴露bus刷新配置的端点
    web:
      exposure:
        include: 'bus-refresh'  #凡是暴露监控、刷新的都要有actuator依赖，bus-refresh就是actuator的刷新操作
```
### 客户端配置
```
spring:
  application:
    name: config-client
  cloud:
    #Config客户端配置
    config:
      label: master #分支名称
      name: config #配置文件名称
      profile: dev #读取后缀名称 上述3个综合：master分支上config-dev.yml的配置文件被读取 http://config-3344.com:3344/master/config-dev.yml
      uri: http://localhost:3344 #配置中心地址
  #rabbit相关配置 15672是web管理界面的端口，5672是MQ访问的端口
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest

#服务注册到eureka地址
eureka:
  client:
    service-url:
      defaultZone: http://localhost:7001/eureka

#暴露监控端点
management:
  endpoints:
    web:
      exposure:
        include: "*"
```

### config刷新方式：
只需要调用一次服务端的刷新连接即可

## SpringCloud Sleuth分布式链路跟踪
下载运行zipkin：zipkin-server-2.12.9-exec.jar
http://localhost:9411/zipkin/

### 服务端配置
```
<!--包含了sleuth+zipkin-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-zipkin</artifactId>
        </dependency>
```
```
spring:
  application:
    name: cloud-order-server
  # zipkin/sleuth链路跟踪
  zipkin:
    base-url: http://localhost:9411
  sleuth:
    sampler:
      # 采样值介于0到1之间,1表示全部采集
      probability: 1
```
**服务调用了以后才能看到链路追踪**

## SpringCloud Stream消息驱动

屏蔽底层消息中间件的差异，降低切换成本，统一消息的编程模型
### 标准流程
- Binder
	很方便的连接中间件，屏蔽差异
- Channel
	通道，是队列Queue的一种抽象，在消息通讯系统中就是实现存储和转发的媒介，通过Channel对队列进行配置
- Source和Sink
	简单的可以理解为参照对象是Spring Cloud Stream 自身，从Stream发布消息就是输出，接受消息就是输入

### productor配置
**destination可以认为是kafka中的topic**
```
spring:
  application:
    name: cloud-stream-provider
  cloud:
    stream:
      binders: # 在此处配置要绑定的rabbitMQ的服务信息
      defaultRabbit: # 表示定义的名称，用于binding的整合
          type: rabbit # 消息中间件类型
          environment: # 设置rabbitMQ的相关环境配置
            spring:
              rabbitmq:
                host: localhost
                port: 5672
                username: guest
                password: guest
      bindings: # 服务的整合处理
        ouput: # 这个名字是一个通道的名称
          destination: studyExchange # 表示要使用的exchange名称定义
          content-type: application/json # 设置消息类型，本次为json，文本则设为text/plain
          binder: defaultRabbit # 设置要绑定的消息服务的具体设置

eureka:
  client:
    service-url:
      defaultZone: http://eureka7001.com:7001/eureka
  instance:
    lease-renewal-interval-in-seconds: 2 # 设置心跳的间隔时间，默认30
    lease-expiration-duration-in-seconds: 5 # 超过5秒间隔，默认90
    instance-id: send-8801.com # 主机名
    prefer-ip-address: true # 显示ip
```
```
@EnableBinding(Source.class)
public class MessageProviderImpl implements IMessageProvider {
    /**
     * 消息发送管道
     */
    @Resource
    private MessageChannel output;
    @Override
    public String send() {
        String serial = UUID.randomUUID().toString();
        output.send(MessageBuilder.withPayload(serial).build());
        System.out.println("serial = " + serial);
        return null;
    }
}
```
### consumer配置
**使用group属性来解决重复消费**
```
spring:
  application:
    name: cloud-stream-consumer
  cloud:
    stream:
      binders: # 在此处配置要绑定的rabbitMQ的服务信息
        defaultRabbit: # 表示定义的名称，用于binding的整合
          type: rabbit # 消息中间件类型
          environment: # 设置rabbitMQ的相关环境配置
            spring:
              rabbitmq:
                host: localhost
                port: 5672
                username: guest
                password: guest
      bindings: # 服务的整合处理
        input: # 这个名字是一个通道的名称
          destination: studyExchange # 表示要使用的exchange名称定义
          content-type: application/json # 设置消息类型，本次为json，文本则设为text/plain
          binder: defaultRabbit # 设置要绑定的消息服务的具体设置
          group: atguiguA
eureka:
  client:
    service-url:
      defaultZone: http://eureka7001.com:7001/eureka
  instance:
    lease-renewal-interval-in-seconds: 2 # 设置心跳的间隔时间，默认30
    lease-expiration-duration-in-seconds: 5 # 超过5秒间隔，默认90
    instance-id: receive-8803.com #主机名
    prefer-ip-address: true # 显示ip
```
```
@Component
@EnableBinding(Sink.class)
public class ReceiveMessageListenerController {
    @Value("${server.port}")
    private String serverPort;

    @StreamListener(Sink.INPUT)
    public void input(Message<String> message){
        System.out.println("消费者2号，------->接收到的消息： "+message.getPayload()+"\t port: "+serverPort);
    }
}
```
