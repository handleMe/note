# spring

## IOC
IOC就是控制反转，指创建对象的控制权转移给Spring框架进行管理，并由Spring根据配置文件去创建实例和管理各个实例之间的依赖关系，对象与对象之间松散耦合，也利于功能的复用。DI依赖注入，和控制反转是同一个概念的不同角度的描述，即 应用程序在运行时依赖IoC容器来动态注入对象需要的外部依赖
Spring的IOC有三种注入方式 ：构造器注入、setter方法注入、根据注解注入

BeanDefinition：描述bean配置情况的实例
### 生命周期
解析配置 - 》 反射创建实例 -》 set属性值 -》如果实现了*aware接口，就调用相关的方法 - 》如果实现了BeanPostProcess，就调用postProcessBeforeInitialization -》 如果指定了init-method就调用它 -》如果实现了BeanPostProcess，就调用postProcessAfterInitialization() -》如果Bean实现了DisposableBean接口，执行destroy()方法 -》如果指定了destroy-method 调用它
1.Bean容器找到配置文件中Spring Bean的定义。
2.Bean容器利用Java Reflection API创建一个Bean的实例。
3.如果涉及到一些属性值，利用set()方法设置一些属性值。
4.如果Bean实现了BeanNameAware接口，调用setBeanName()方法，传入Bean的名字。
5.如果Bean实现了BeanClassLoaderAware接口，调用setBeanClassLoader()方法，传入ClassLoader对象的实例。
6.如果Bean实现了BeanFactoryAware接口，调用setBeanClassFacotory()方法，传入ClassLoader对象的实例。
7.与上面的类似，如果实现了其他*Aware接口，就调用相应的方法。

8.如果有和加载这个Bean的Spring容器相关的BeanPostProcessor对象，执行postProcessBeforeInitialization()方法。
9.如果Bean实现了InitializingBean接口，执行afeterPropertiesSet()方法。
10.如果Bean在配置文件中的定义包含init-method属性，执行指定的方法。

11.如果有和加载这个Bean的Spring容器相关的BeanPostProcess对象，执行postProcessAfterInitialization()方法。
12.当要销毁Bean的时候，如果Bean实现了DisposableBean接口，执行destroy()方法。
13.当要销毁Bean的时候，如果Bean在配置文件中的定义包含destroy-method属性，执行指定的方法。

### 生命周期另一种描述解释
1：实例化一个ApplicationContext的对象；
2：调用bean工厂后置处理器完成扫描；
3：循环解析扫描出来的类信息；
4：实例化一个BeanDefinition对象来存储解析出来的信息；
5：把实例化好的beanDefinition对象put到beanDefinitionMap当中缓存起来，以便后面实例化bean；
6：再次调用bean工厂后置处理器；
7：当然spring还会干很多事情，比如国际化，比如注册BeanPostProcessor等等，如果我们只关心如何实例化一个bean的话那么这一步就是spring调用finishBeanFactoryInitialization方法来实例化单例的bean，实例化之前spring要做验证，需要遍历所有扫描出来的类，依次判断这个bean是否Lazy，是否prototype，是否abstract等等；
8：如果验证完成spring在实例化一个bean之前需要推断构造方法，因为spring实例化对象是通过构造方法反射，故而需要知道用哪个构造方法；
9：推断完构造方法之后spring调用构造方法反射实例化一个对象；注意我这里说的是对象、对象、对象；这个时候对象已经实例化出来了，但是并不是一个完整的bean，最简单的体现是这个时候实例化出来的对象属性是没有注入，所以不是一个完整的bean；
10：spring处理合并后的beanDefinition；
11：判断是否支持循环依赖，如果支持则提前把一个工厂存入singletonFactories——map；
12：判断是否需要完成属性注入
13：如果需要完成属性注入，则开始注入属性
14：判断bean的类型回调Aware接口
15：调用生命周期回调方法
16：如果需要代理则完成代理
17：put到单例池——bean完成——存在spring容器当中

### spring循环依赖：
spring当中默认支持单例状态下的循环依赖
## AOP

AOP，一般称为面向切面，作为面向对象的一种补充，用于将那些与业务无关，但却对多个对象产生影响的公共行为和逻辑，抽取并封装为一个可重用的模块
AOP实现的关键在于 代理模式，AOP代理主要分为静态代理和动态代理


## spring的AOP顺序
### AOP常用注解：

#### @Pointcut指定通知范围
	- execution：用于匹配方法执行的连接点
	- within：用于匹配指定类型内的方法执行
	- this：用于匹配当前AOP代理对象类型的执行方法；注意是AOP代理对象的类型匹配，这样就可能包括引入接口也类型匹配
	- target：用于匹配当前目标对象类型的执行方法；注意是目标对象的类型匹配，这样就不包括引入接口也类型匹配
	- args：用于匹配当前执行的方法传入的参数为指定类型的执行方法
	- @within：用于匹配所以持有指定注解类型内的方法
	- @target：用于匹配当前目标对象类型的执行方法，其中目标对象持有指定的注解
	- @args：用于匹配当前执行的方法传入的参数持有指定注解的执行
	- @annotation：用于匹配当前执行方法持有指定注解的方法
	- bean：Spring AOP扩展的，AspectJ没有对于指示符，用于匹配特定名称的Bean对象的执行方法

#### 通知时机：
- @Before
- @After
- @AfterReturning
- @AfterThrowing
- @Around

### spring4-spring5的AOP全部执行顺序，有哪些坑？你遇到过吗？
springboot1->2，spring也从4升级到了5，而spring4和5 的AOP的执行顺序是不同的
#### spring4
正常执行顺序：
- @Around环绕前置通知（即调用proceed方法之前）
- @Before前置通知
- 程序本体执行
- @Around后置通知（即调用proceed方法之后）
- @After后置通知
- @AfterReturning返回后通知

异常的执行顺序：
- @Around环绕前置通知（即调用proceed方法之前）
- @Before前置通知
- @After后置通知
- @AfterThrowing异常通知

总结：
执行顺序类似

```
try{
	@Before
	method.invoke(obj, args);
	@AfterReturning
}catch(Exception e){
	@AfterThrowing
}finally{
	@After
}
```
#### spring5
正常执行顺序：
- @Around环绕前置通知（即调用proceed方法之前）
- @Before前置通知
- 程序本体执行
- @AfterReturning返回后通知
- @After后置通知
- @Around后置通知（即调用proceed方法之后）

异常的执行顺序：
- @Around环绕前置通知（即调用proceed方法之前）
- @Before前置通知
- @AfterThrowing异常通知
- @After后置通知


## 事务
- 原子性：一组操作不可分割
- 一致性：事务必须使数据库从一个一致性状态变换到另一个一致性状态
- 隔离性：不同事务之间数据隔离
- 持久性：事务提交以后，数据的修改是永久的

### 说明
- 脏读：一个事务获取了其他事务未提交的值；事务A更新了某个值，由于某种原因回滚了数据，事务B在回滚之前读取了这个临时的值，就是脏数据
- 不可重复度：一个事务在两次查询同一条数据获得了不同的值；事务A查询了数据X，在A未提交时，事务B修改了数据X，事务A再次读取数据X的时候，X的状态已经发生变化
- 幻读：事务A读取某个范围的数据，事务B在这个范围中插入了一条数据，事务A再次读取时，产生了幻读行

### 隔离级别：
- ISOLATION_DEFAULT：使用后端数据库默认的隔离级别，Mysql默认采用的REPEATABLE_READ隔离级别；Oracle默认采用的READ_COMMITTED隔离级别。
- ISOLATION_READ_UNCOMMITTED：最低的隔离级别，允许读取尚未提交的数据变更，可能会导致脏读、幻读或不可重复读。
- ISOLATION_READ_COMMITTED：允许读取并发事务已经提交的数据，可以阻止脏读，但是幻读或不可重复读仍有可能发生
- ISOLATION_REPEATABLE_READ：对同一字段的多次读取结果都是一致的，除非数据是被本身事务自己所修改，可以阻止脏读和不可重复读，但幻读仍有可能发生。幻读是指某个事物在读取某个范围内的记录时，另外一个事物又在该范围内插入了新的记录，当之前的事物再次读取该范围的记录时，会产生幻读行。**mysql的InnoDB通过多版本并发控制（MVCC）解决了幻读的问题；所以使用InnoDb在这个级别就不会产生幻读了**
- ISOLATION_SERIALIZABLE：最高的隔离级别，完全服从ACID的隔离级别。所有的事务依次逐个执行，这样事务之间就完全不可能产生干扰，也就是说，该级别可以防止脏读、不可重复读以及幻读。但是这将严重影响程序的性能。通常情况下也不会用到该级别。
### 传播行为：

#### 支持当前事务
- PROPAGATION_REQUIRED：如果当前存在事务，则加入该事务；如果当前没有事务，则创建一个新的事务。
- PROPAGATION_SUPPORTS： 如果当前存在事务，则加入该事务；如果当前没有事务，则以非事务的方式继续运行。
- PROPAGATION_MANDATORY： 如果当前存在事务，则加入该事务；如果当前没有事务，则抛出异常。（mandatory：强制性）。
##### 不支持当前事务的情况：
- PROPAGATION_REQUIRES_NEW： 创建一个新的事务，如果当前存在事务，则把当前事务挂起。
- PROPAGATION_NOT_SUPPORTED： 以非事务方式运行，如果当前存在事务，则把当前事务挂起。
- PROPAGATION_NEVER： 以非事务方式运行，如果当前存在事务，则抛出异常。
#### 其他情况：
- PROPAGATION_NESTED： 如果当前存在事务，则创建一个事务作为当前事务的嵌套事务来运行；如果当前没有事务，则该取值等价于PROPAGATION_REQUIRED。

### 示例
A调用B
案例1，B的事务级别定义为REQUIRED,
  1、如果A已经起了事务，这时调用B，会共用同一个事务，如果出现异常，A和B作为一个整体都将一起回滚。
  2、如果A没有事务，B就会为自己分配一个事务。A中是不受事务控制的。如果出现异常，B不会引起A的回滚。
** 挂起**
案例2，A的事务级别REQUIRED，B的事务级别REQUIRES_NEW，
A调用B，A所在的事务就会挂起，B会起一个新的事务。

1、如果B已经提交，那么A失败回滚，B是不会回滚的。
2、如果B失败回滚，如果他抛出的异常被A的try..catch捕获并处理，A事务仍然可能提交；如果他抛出的异常未被A捕获处理，A事务将回滚。
** 嵌套 **
案例3，A的事务级别为REQUIRED，B的事务级别为NESTED，
A调用B的时候，A所在的事务就会挂起，B会起一个新的子事务并设置savepoint
1、如果B已经提交，那么A失败回滚，B也将回滚。
2、如果B失败回滚，如果他抛出的异常被A的try..catch捕获并处理，A事务仍然可能提交；如果他抛出的异常未被A捕获处理，A事务将回滚。

## 源码查看
以ClassPathXmlApplicationContext为例：
1. setConfigLocations
- resolvePath()循环解析每个配置文件
- - getEnvironment()获取运行时环境{含有profiles和properties}
- - resolveRequiredPlaceholders
- - - createPlaceholderHelper 创建占位符辅助对象
- - - doResolvePlaceholders 解析占位符{指配置文件名称上使用的占位符}
2. **refresh**
- applicationStartup.start 创建默认的StartupStep：DefaultStartupStep
    - - prepareRefresh 准备上下文
        - - - initPropertySources 默认没有任何操作，留作扩展使用
        - - - getEnvironment().validateRequiredProperties()验证属性是否可解析
- 创建ConfigurableListableBeanFactory
  
    - - loadBeanDefinitions - > loadBeanDefinitions解析配置
- prepareBeanFactory准备beanFactoory
- registerBeanPostProcessors 创建beanPostProcessor，但不会调用
- initMessageSource 初始化国际化
- initApplicationEventMulticaster 初始化事件多播器
- onRefresh()初始化特定的bean
- registerListeners 注册监听器
- **finishBeanFactoryInitialization 初始化单例非懒加载的bean**
```
- - **beanFactory.preInstantiateSingletons();**
- - - getBean ->
- - -  doGetBean
		getSingleton 直接去一级缓存获取
		 createBean->
		  doCreateBean -> 
			createBeanInstance 获得BeanWrapper包装实例
                instantiateBean 
                    instantiate筛选构造器
                      调用BeanUtils.instantiateClass使用筛选出来的构造器初始化bean
                使用BeanWrapper包装实例
                initBeanWrapper
            放入三级缓存addSingletonFactory（beanName = xxBeanFactory）
			populateBean 填充实例；属性注入
				
				inject
				  InjectedElement.inject
				  	resolveDependency
				  		doResolveDependency
				  		 findAutowireCandidates
				  		  addCandidateEntry
				  		descriptor.resolveCandidate获取注入对象；注入对象初始化成功就放入一级缓存
				  			其中又调用getBean对象开始创建
				  			创建成功后放入三级缓存
				  			开始为B注入属性A；这时候从三级缓存中获取到B，然后把A放入二级缓存，注入属性，实例化完成后，将B放入一级缓存
				  	registerDependentBeans
				  		registerDependentBean
				  	field.set(bean, value);注入获取的对象
			initializeBean 初始化bean；调用配置的init以及钩子函数
				PostBeanProcessor的前置方法调用
				initializeBean调用初始化方法
				postProcessAfterInitialization：aop也是通过实现了一个BeanPostProcessor来完成对bean的代理
					wrapIfNecessary
					查找切点
						createProxy
							getProxy
								createAopProxy根据类属性或者配置选择jdk或者cglib代理，生成字节码、获取代理对象
								
			
				完成后，将目标bean放入一级缓存
- finishRefresh
```
## 钩子函数
1. Aware接口族，Spring中提供了各种Aware接口，方便从上下文中获取当前的运行环境，常见的包括：BeanFactoryAware，BeanNameAware，ApplicationContextAware，EnvironmentAware，BeanClassLoaderAware等。
- BeanFactoryAware接口：通过它可以拿到Spring容器中的Bean。通过调用getBean方法就可以拿到特定的Bean，使用BeanFactoryAware的好处是，需要动态获取具体Bean的时候很方便
- ApplicationContextAware接口：类似于BeanFactoryAware，区别就是ApplicationContext是BeanFactory的区别
- EnvironmentAware接口：凡注册到Spring容器内的bean，实现了EnvironmentAware接口重写setEnvironment方法后，在工程启动时可以获得application.properties的配置文件配置的属性值

2. InitializingBean接口和DisposableBean接口
初始化和销毁方法接口
3. BeanPostProcessor和BeanFactoryPostProcessor接口族
- BeanFactoryPostProcessor可以对bean的定义进行处理，Spring容器允许BeanFactoryPostProcessor在容器实际实例化任何其它的bean之前读取配置元数据，并有可能修改它。可以配置多个BeanFactoryPostProcessor，执行次序可以配置
- 实现BeanPostProcessor接口可以在Bean(实例化之后)初始化的前后做一些自定义的操作

## springmvc


## springboot
主配置类注解@SpringBootApplication：

- @SpringbootConfiguration：springboot配置类：标识这是一个springboot的配置类
- - 底层是@Configuration
- EnableAutoConfiguration：开启自动配置功能
- @ComponentScan：
- - 底层是：AutoConfigurationPackage默认扫描主配置类所在包及以下所有子包扫描到容器中
- - - 底层是使用@Import（AutoConfigurationImportSelector）：导入组件，扫描组件的自动配置类，进行配置auto configuration classes found in META-INF/spring.factories.
	- 扫描资源过程： AutoConfigurationImportSelector.selectImports
           getAutoConfigurationEntry
            getCandidateConfigurations
              loadFactoryNames ->loadSpringFactories
                扫描加载META-INF/spring.factories的配置
                每一个XXXAutoConfiguration都作为一个容器的组件加载
                多与ConditionalXXX注解联合使用，即：满足条件时此配置类才生效，属于@Condition注解的衍生注解，
                EnableConfigurationProperties则是引入的配置属性类
                
### 启动原理

1. 创建SpringApplication对象
	调用initialize（source），source就是主配置类
	setInitializers 从类路径下找到META-INF/spring.factories配置的所有ApplicationContextInitializer
	setListeners从类路径下找到META-INF/spring.factories配置的所有ApplicationListener
	判断主配置类（含有main方法的）
2. 调用run方法
	- 获取SpringApplicationRunListeners//从类路径下META-INF/spring.factories中获取
	- listener.starting();回调所有的listener的starting方法
	- 封装命令行参数
	- 准备环境，ConfigurableEnvironment environment = prepareEnvironment
		- 完成后回调SpringApplicationRunListener.enviromentPrepared，表示环境准备完成
		- 创建IOC容器
		- 准备上下文prepareContext
			- 将enviroment保存到context中
			- applyInitializer回调之前保存的所有的ApplicationContextInitializer的initialze（）方法
			- 回调所有的SpringApplicationRunListener的contextPrepared方法
			- 最后回调所有的SpringApplicationRunListener的contextLoaded方法
		- 刷新容器refreshContext：就是IOC容器初始化，就是调用refresh方法；如果是web应用，还会创建嵌入式的tomcat
		- afterRefresh
			- callRunner
				- 从IOC容器中获取所有的ApplicationRunner，调用的callRunner方法
				- 从IOC容器中获取所有的CommandLineRunner，调用的callRunner方法
		- listeners.finish调用所有的SpringApplicationRunListener的callFinishedListener方法
		- 最后返回IOC容器


- 准备环境
	- 执行ApplicationContextInitializer.initialize()
	- 监听器SpringApplicationRunListener回调contextPrepared
	- 加载主配置类定义信息
	- 监听器SpringApplicationRunListener回调contextLoaded
- 刷新启动IOC容器；
	- 扫描加载所有容器中的组件
	- 包括从META-INF/spring.facories中获取所有EnableAutoConfiguration组件
- 回调容器中所有的ApplicationRunner、CommandLineRunner的run方法
- 监听器SpringApplicationRunListener回调finished
#### 事件监听机制
- 配置在META-INF/spring.factorie中的
ApplicationContextInitializer通过实现这个接口<指定具体初始化器>来对具体的初始化器进行扩展
SpringApplicationRunListener通过实现这个接口来扩展此监听器，需要一个有参构造，可以参照自带的实现类
**注入为组件就能生效**

- 放在IOC容器中的
ApplicationRunner通过扩展此接口来扩展
CommandLineRunner通过扩展此接口来扩展
这两个接口都只有一个run方法，参数是启动类的args
他们的实现类需要**在类路径下的META-INF的spring.factories中进行配置并且注入为组件才能生效**

### 运行流程

### 自动配置原理
#### springboot自定义starter
- 配置类：
使用@Configuration指定
使用@ConditionXXX是定生效条件
使用@AutoConfigureOrder指定配置类的顺序
使用@ConfigurationProperties来指定属性配置类
@EnableConfigurationPropertoes--让xxxProperties生效
在类中使用@Bean注入组件

- 将配置类配置在：META-INF/spring.factories中
**启动器只用来做依赖导入**

所有starter都需要引入依赖spring-boot-starter