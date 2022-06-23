# JUC java.util.concurrent

多线程实现的四种方式

- 继承Thread
- 实现Runnable接口
- 实现Callable接口
有返回值的线程实现
```
public class CallableDemo {
    public static void main(String[] args) throws Exception {
        FutureTask<Integer> ft = new FutureTask<>(new myCall());
        Thread t1 = new Thread(ft, "t1");
        t1.start();

        new Thread(ft, "t2").start();
        new Thread(ft, "t3").start();

        System.out.println(ft.get());
        /*
         * ft.get()该方法要求获取返回结果，如果线程没有执行完成，就会阻塞当前线程，等待结果
         */
    }
}

class myCall implements Callable {

    @Override
    public Integer call() throws Exception {
        System.out.println("call...");
        return 1024;
    }
}

```

- 使用线程池

## java对象构成
//小端存储，实际上是从后往前看的（mac除外）

- 对象头：大小->96bit；数组对象128bit
	- Mark Word：64位下共64bit：unused：25；hash：31；unused：1；age：4；biased_lock：1；lock：2
		- lock：锁状态；总共只有两位，而对象状态有5个，其中偏向锁是使用lock+biased_lock一同表示
		- biased_lock：表示是否是偏向锁
		- age：年龄，4bit，最大表示15，所以对象进入老年代的阈值只能<=15
		- hash：只有调用了hashcode的时候才会计算，不计算的时候是空的，即0
```
偏向锁状态：54bit当前线程指针+2bit验证偏向锁有效性的时间戳+1unused+4bit年龄+1bit偏向锁位+2bit锁标志
轻量级锁状态：指向栈中LockRecord的指针+2bit锁标志位
重量级锁：指向互斥量（重量级锁）的指针+2bit锁标志位
GC标记信息：CMS过程用到的标记信息+2bit锁标志
```
	- klass pointer
	    - 指向类的指针 ->XXClass.class
	    - 数组长度（只有数组对象才有），没有的时候是0
- 实例数据
- 对齐补充字节

## synchronized
重量级的锁，排他、可重入、非公平
--- 实现的汇编指令：lock comchg即：lock compare change

```
//实例方法，锁住的是类的实例对象
synchronized void method(){
}
//静态方法，锁住的是类对象
static synchronized void method(){
}
//同步代码快，锁住类对象
synchronized(this){
}
//同步代码块，锁住类的类对象
synchronized (Demo.class){
}
```
基于wait/notify方法结合synchronized关键字来实现线程通知
### 对象的状态
- 无状态 0 01 new出来的时候
- 偏向锁 1 01 如果设置偏向锁启动延迟为0；创建出来的对象就是带有偏向锁的，此时是匿名偏向的
- 轻量锁    00
- 重量锁    10
- GC标记状态 11
### 锁升级
无锁->偏向锁->轻量级锁->重量级锁
禁用偏向锁：无锁->轻量级锁->重量级锁
偏向锁延时=0：（匿名）偏向锁->轻量级锁->重量级锁
偏向锁竞争严重时也会直接晋升重量级锁

#### 偏向锁
为什么有偏向锁：多数情况下，只有一个线程在使用资源
偏向锁在1.6之后是默认开启的，但是有几秒延时，应用程序启动几秒之后才激活
使用```-XX:BiasedLockingStartupDelay=0```来关闭延时；如果确定业务使用过程中，锁通常处在竞争状态下，可以通过```-XX:-UseBiasedLocking=false```来关闭偏向锁
**无锁->偏向锁**

1. 当线程第一次访问同步代码块，虚拟机会把对象头中的标志位设置为1，即偏向锁
2. 同时使用CAS操作吧这个锁的线程ID记录在对象的MarkWork中，占用之前没有使用的25位+hash的31位；ThreadID54bit，Epoch2bit，**持有偏向锁的线程以后每次进入这个锁相同的同步块中，虚拟机度可以不进行同步操作，偏向锁的效率高**
##### 偏向锁的获取撤销
**获取：**

1. 首先获取锁 对象的 Markword，判断是否处于可偏向状 态。（biased_lock=1、且 ThreadId 为空）

2. 如果是可偏向状态，则通过 CAS 操作，把当前线程的 ID 写入到 MarkWord a) 如果 cas 成功，那么 markword 就会变成这样。 表示已经获得了锁对象的偏向锁，接着执行同步代码 块 b) 如果 cas 失败，说明有其他线程已经获得了偏向锁， 这种情况说明当前锁存在竞争，需要撤销已获得偏向 锁的线程，并且把它持有的锁升级为轻量级锁（这个 操作需要等到全局安全点，也就是没有线程在执行字 节码）才能执行

3. 如果是已偏向状态，需要检查 markword 中存储的 ThreadID 是否等于当前线程的 ThreadID a) 如果相等，不需要再次获得锁，可直接执行同步代码 块 b) 如果不相等，说明当前锁偏向于其他线程，需要撤销 偏向锁并升级到轻量级

  **撤销：**

4. 原获得偏向锁的线程如果已经退出了临界区，也就是同 步代码块执行完了，那么这个时候会把对象头设置成无锁状态，并且争抢锁的线程可以基于 CAS 重新偏向当前线程

5. 如果原获得偏向锁的线程的同步代码块还没执行完，处 于临界区之内，这个时候会把原获得偏向锁的线程升级 为轻量级锁后继续执行同步代码块

===================================

- 偏向锁的撤销必须等待全局安全点（所有线程暂停）
- 暂停拥有偏向锁的线程，判断对象是否处于被锁定状态
- 撤销偏向锁，恢复到无锁或者升级到轻量级锁
** 偏向锁的优点**
持有偏向锁的线程以后每次进入这个锁相同的同步块中，虚拟机度可以不进行同步操作，偏向锁的效率高；所以一个线程不断进入同步代码快的时候，比较高效，如果有线程竞争的时候，会升级成为轻量级锁，升级过程中需要等待去全局安全点，代价较高，所以需要在合适的场景下使用

#### 轻量级锁（自旋锁）
偏向锁->轻量级锁
**只要有另外的线程来竞争就会升级**

1. 构建LockRecord对象
LockRecord
	- displaced hdr
		- 复制hashcode、年龄、原有锁标记01
	- owner：指向对象
2. 除锁标记为其他MarkWord位置记录存储LockRecord指针
3. 第二步操作使用CAS，成功后修改对象的锁标记为00
4. 如果失败则判断当前对象的Mark Word是否指向当前线程的栈帧，如果是则表示当前线程已经持有当前对象的锁，则直接执行同步代码块；否则只能说明该锁对象已经被其他线程抢占了，这时轻量级锁需要膨胀为重量级锁，锁标志位变成10，后面等待的线程将会进入阻塞状态。
##### 偏向锁一定优于轻量级锁吗？
不一定，因为如果明确知道多个线程竞争的话，直接上自旋锁比较好，因为偏向锁撤销的话，需要等待全局安全点
#### 重量级锁（互斥锁）
重量级锁是依赖对象内部的monitor锁来实现的，而monitor又依赖操作系统的MutexLock(互斥锁)来实现的，所以重量级锁也被成为互斥锁。
重量级锁真正阻塞线程

轻量级锁->重量级锁
**
1.6之前，有线程自旋超过10次或者超过CPU核数一半的线程在等待，就升级
1.6之后进行自适应自旋
**
轻量级锁的锁释放逻辑其实就是获得锁的逆向逻辑，通过 CAS 操作把线程栈帧中的 LockRecord 替换回到锁对象的 MarkWord 中，如果成功表示没有竞争。如果失败，表示 当前锁存在竞争，那么轻量级锁就会膨胀成为重量级锁

#### 锁消除（v1.8默认开启）

需要开启逃逸分析，1.8默认是开启的

值即时编译器JIT在运行时，对一些代码上要求同步，但是被检测到不可能存在共享数竞争的锁进行所消除，主要判断依据是**逃逸分析和数据支持**
如果一段代码中，堆上所有数据都不会逃逸出去从而被其他线程访问到，那就可以把他们当做栈上的数据对待，认为是线程私有的，同步加锁自然无需进行
**比如使用StringBuffer连续append三个字符串：**
StringBuffer的append ( ) 是一个同步方法，锁就是this也就是(new StringBuilder())。虚拟机发现它的动态作用域被限制在concatString( )方法内部。也就是说, new StringBuilder()对象的引用永远不会“逃 逸”到concatString ( )方法之外，其他线程无法访问到它，因此，虽然这里有锁，但是可以被安全地消除掉，在即时编译之后，这段代码就会忽略掉所有的同步而直接执行了

#### 锁粗化
一连串细小的操作都使用同一个对象进行加锁，将同步代码块的范围放大，放到这串代码块的外面，只加一次锁

比如：在一个循环中进行StringBuffer的append ( )
#### 优化
- **降低范围：同步代码块尽量短**
- **降低锁粒度：将一个锁拆分为多个锁，提高并发度**：比如1.7的ConcurrentMap使用的分段锁
- 
### monitor对象
```
ObjectMonitor(){
	_header
	_count
	_waiters
	_recursions：线程重入次数
	_object：存储锁住的对象
	_owner：标识拥有该monitor的线程
	_waitSet：处于wait状态的线程，会被加入_waitSet
	_waitSetLock
	_Responsible
	_succ
	_cxq：多线程竞争的单向列表
	FreeNext
	_EntryList：处于等待锁lock状态的线程会被加入到这个列表
	_SpinFreq
	_spinClock
	OwnerIsThread
}
```

## Condition
java中的等待通知机制，可以通过Condition和Lock结合实现，Condition可以实现比较精确的通知机制；依赖于Lock
/**
 * 实现多线程之间按顺序调用，实现A、B、C三个线程启动，实现：
 * AA打印5次，BB打印10次，CC打印15次
 * 紧接着
 * AA打印5次，BB打印10次，CC打印15次
 * 循环10次
 */
```
	private int number = 1;
    private Lock lock = new ReentrantLock();
    private Condition c1 = lock.newCondition();
    private Condition c2 = lock.newCondition();
    private Condition c3 = lock.newCondition();
    public void print5(int i){
        lock.lock();
        try{
            //判断
            while(number != 1){
                c1.await();
            }
            print(5, i);
            number = 2;
            c2.signal();//通知指定的线程
        }catch(Exception e){
            e.printStackTrace();
        }finally {
            lock.unlock();
        }
    }
	public void print10(int i){
        lock.lock();
        try{
            //判断
            while(number != 2){
                c2.await();
            }
            print(10, i);
            number = 3;
            c3.signal();
        }catch(Exception e){
            e.printStackTrace();
        }finally {
            lock.unlock();
        }
    }
    public void print15(int i){
        lock.lock();
        try{
            //判断
            while(number != 3){
                c3.await();
            }
            print(15, i);
            number = 1;
            c1.signal();
        }catch(Exception e){
            e.printStackTrace();
        }finally {
            lock.unlock();
        }
    }
```


## JMM java内存模型
&emsp;&emsp;抽象的概念，描述一种规范
- 可见性
- 原子性
- 有序性
      使用atomic包下的类保证原子性
- 主内存：java虚拟机规定所有的变量(不是程序中的变量)都必须在主内存中产生
- 工作内存：java虚拟机中每个线程都有自己的工作内存
### volitale关键字

--- 使用的关键汇编指令 lock addl

- 可见性
- 不保证原子性
- 禁止指令重排（有序性）
### 内存屏障
&emsp;&emsp;作用有两个：
-  一是保证特定操作的执行顺序；（无论什么指令都不能与Memory Barrier指令重新排序）
-  二是保证某些变量的内存可见性；（强制刷出各种CPU的缓存数据），volitale变量在进行写操作的时候，会在操作后加入一条store屏障指令，将工作内存中的共享变量值刷新到主内存；对Volitale变量进行读取操作的时候，会在读操作前加入一条load屏障指令，从主内存读取当前的共享变量

#### JSR内存屏障
Load 写 Store读；这是jvm的一个规范，不是实现
- LoadLoad：load1；LoadLoad；load2；load1读取完毕，后续操作才能继续操作这个数据
- StoreLoad：store1；StoreLoad；load2；在load2及后续操作之前，保证store1的写入对所有处理器可见
- LoadStore：load1；LoadStore；store2；在store2及后续操作之前，保证load1要读取的数据被读取完毕
- StoreStore： Store1；StoreStore； Store2； Store2及后续写入操作执行前，保证store1写入操作对其他处理器可见
#### 指令重排
计算机执行程序时，为了提高性能，编译器和处理器经常会对指令做重排，一般分为三种：
源代码=>编译器优化的重排=>指令并行的重排=>内存系统的重排=>最终执行命令
指令重排一定会考虑数据依赖性

### CAS
compare and swap
和synchronized相比，提高了并发性，但是可能导致CPU消耗过大
缺点：

- 循环时间长，开销大
- 只能保证一个共享变量的原子操作
- 存在ABA问题
### Unsafe类
CAS的核心类，JVM会帮我们实现CAS的汇编指令实现原子性
### ABA问题 
针对CAS产生的ABA问题，使用加版本的方式避免
AtomicStampedReference是加了版本的原子引用包装类

### 原子引用包装
AtomicReference<>
```
AtomicReference<User> atmicUser = new AtomicReference<>();
User admin = new User();
atmicUser.set(admin);
atomicUser.compareAndSet(amdin, new User());
```
### AtomicStampedReference<>
带有版本控制的原子引用，可以解决ABA问题


## java的锁
### 公平锁和非公平锁
公平锁：多个线程按照申请锁的顺序来获取锁；并发环境中，线程获取锁的时候，会先查看这个锁维护的等待队列，如果为空，或者当前线程是等待队列的第一个，就占有锁，否则加入队列等待，按照FIFO的规则获取
非公平锁：指多个线程获取锁的顺序并不是按照申请锁的顺序，有可能后申请的线程先获取锁，高并发的时候，有可能造成优先级反转或者饥饿现象；线程会直接尝试获取锁，如果获取失败，就采用类似公平锁的方式等待获取
** ReentrntLock，默认情况是非公平锁，传入参数true是公平锁，不传，默认false，非公平锁优点是吞吐量比公平锁大**

### 可重入锁（递归锁）
同一个线程外层函数 获取锁之后，内层递归函数仍然能够获取该锁的代码；
在同一个线程的外层方法获取锁的时候，在进入内层方法会自动获取锁
就是说，线程可以进入任何一个它已经拥有的锁所同步着的代码块
第一个加锁的代码块中调用了另外的一个加锁方法，用的是同一把锁；
**ReentrntLock和synchronized是可重入锁**
**最大的作用是避免死锁**

### 自旋锁 spinLock
尝试获取锁的线程不会立即阻塞，而是采用循环的方式尝试获取锁，减少了线程上下文切换的消耗，缺点是消耗CPU资源
### 独占锁（写锁）/共享锁（读锁）/互斥锁

#### 读写锁：ReadWriteLock ：实现ReentrantReadWriteLock
多个线程同时读取一个资源没有问题，为了满足并发，读取共享资源可以同时进行，但是读和写不能同时进行，写和写也不能同时进行
写操作：原子+独占，整个过程必须是完整的，不能被分割，不能被打断

### CountDownLatch/CyclicBarrier/Semaphore

### 阻塞队列
**当阻塞队列是空的时候，获取元素的操作将会被阻塞
当阻塞队列是满的时候，插入元素的操作将会被阻塞**
ArrayBlockingQueue：由数组组成的有界阻塞队列
LinkedBlockingQueue：链表实现的有界阻塞队列（默认大小是Integer.MAX_VALUE）
SynchronousQueue：不存储元素的阻塞队列，就是单个元素的队列。

PriorityBlockingQueue：支持优先级排序的无界阻塞队列
DelayQueue：使用优先级队列实现的延迟无界阻塞队里
LinkedTransferQueue：链表实现的无界阻塞队列
LinkedBlockingDeque：链表实现的双向阻塞队列


#### 方法说明
|操作：|抛出异常| 特殊值| 阻塞 | 超时|
|:-:|:-:|:-:|:-:|:-:|
|插入：|add(e)|offer(e) true/false|put (e) |offer(e, time, unit)|
|移除：|reomve()|poll() element/null|take()|poll(time, unit)|
|检查：|element()|peek()|不可用|不可用|

#### 传统的生产者消费者--线程通信


### synchronized和lock的区别
1. 构成
synchronized是关键字，属于JVM层面，
monitorenter：底层是通过monitor对象来完成，wait/notify等方法也依赖于monitor对象，只有在同步方法块或方法中才能调用wait/notify等方法
monitorexit
Lock是具体类，是API层面的锁
2. 使用方法：
synchronized不需要用户手动释放锁，系统会自动让线程释放对锁的占用
ReentrantLock则需要用户手动释放锁，如果没有主动释放，就有可能产生死锁
3. 等待是否可以中断
synchronized不可中断，除非抛出异常或者正常运行完成
ReentrantLock可中断，1.设置超时方法，tryLock(long timeout, TimeUnit unit)；	                                                2.lockInterruptibly()放代码块中，调用interrupt()方法可中断
4. 加锁是否公平
synchronized非公平锁
ReentrantLock可以使用fair参数来控制是否是公平锁 默认false非公平
5.锁绑定多个条件Condition
synchronized没有
ReentrantLock用来实现分组唤醒需要唤醒的线程们，可以精确唤醒，而不是像synchronized要么随机唤醒一个线程，要么全部唤醒

#### 生消模式的几种实现方式

### Callable接口
```
	public static void main(String[] args) throws ExecutionException, InterruptedException {
        FutureTask<Integer> ft = new FutureTask<>(new myCall());
        Thread t1 = new Thread(ft, "t1");
        t1.start();

        new Thread(ft, "t3").start();
        Integer x;
        System.out.println(x = ft.get());
        /*
         * ft.get()该方法要求获取返回结果，如果线程没有执行完成，就会阻塞当前线程，等待结果
         */
    }

class myCall implements Callable<Integer> {

    @Override
    public Integer call() throws Exception {
        System.out.println("call...");
        return 9527;
    }
}
```
## 线程池
通过Executor框架实现
核心是ThreadPoolExecutor类实现的

### 快速创建线程池
- Executors.newFixedThreadPool(10)：固定线程数的线程池；核心线程数 = 最大线程数 = 参数
- Executors.newCachedThreadPool()：缓存线程池；核心线程=0；最大线程=Integer.MAX_VALUE
- Executors.newSingleThreadExecutor()：固定线程数的线程池；核心线程数 = 最大线程数 = 1
- Executors.newScheduledThreadPool(5)：定时器线程池；核心线程数=参数；最大线程数=Integer.MAX_VALUE
- Executors.newSingleThreadScheduledExecutor()：定时器线程池；核心线程数=1；最大线程数=Integer.MAX_VALUE

### 参数：
- int corePoolSize：核心线程数，
- int maximumPoolSize：最大线程数
- long keepAliveTime：多余的空闲线程存活时间；当前线程池数量超过corePoolSize时，当空闲时间达到keepAliveTime时，多余的空闲线程会被销毁直至剩下corePoolSize个线程为止
- TimeUnit unit：存活时间的单位
- ```BlockingQueue<Runnable> workQueue```：阻塞队列，存放提交但未被执行的任务 
- ThreadFactory threadFactory：线程工厂，一般使用默认
- RejectedExecutionHandler handler：拒绝策略；工作队列满了的时候，执行的策略

### 线程池运行过程
1.  创建了线程池后，等待提交过来的任务请求
2.  调用execute()方法添加一个请求任务的时候，会做出如下判断：
	2.1 如果正在运行的线程数量小于corePoolSize，那么马上创建线程运行这个任务；
	2.2 如果正在运行的线程数量大于或者等于corePoolSize，那么将这个任务放入队列
	2.3 如果任务队列满了，正在运行的线程数量小于maximumPoolSize。那么创建非核心线程来立刻运行这个任务
	2.4 如果队列满了，且正在运行的线程数量大于等于maximumPoolSize，那么线程池会启动饱和拒绝策略。
3. 当一个线程完成任务的时候，他会从队列中取下一个任务来执行
4. 当一个线程空闲超过一定时间（keepAliveTime）时，线程池会判断：
	4.1 如果当前运行的线程数大于corePoolSize，那么这个线程就会被停掉
所以线程池的所有任务完成后，最终会收缩到corePoolSize大小

### 拒绝策略
JDK内置的拒绝策略：
- AbortPolicy：（默认）直接抛出RejecteExecutionException异常组织系统正常运行
- CallerRunsPolicy：回退策略，谁调用的就回退给谁，main调用就回退给main执行
- DiscardOldestPolicy：丢弃队列中最老的任务, 尝试再次提交当前任务
- DiscardPolicy：默默丢弃无法处理的任务，不予任何处理

### 线程池为什么不允许使用Executors去创建，而是通过ThreadPoolExecutor的方式
这样可以更加明确线程池的运行规则，规避资源耗尽的风险。
弊端如下：
- FixedThreadPool和SingleThreadPool：任务队列长度是Integer.MAX_VALUE，可能会堆积大量的请求，从而导致内存不足OOM
- CachedThreadPool和ScheduledThreadPool，允许创建的线程数量为Integer.MAX_VALUE。可能会创建大量的线程，从而导致OOM

### 线程池参数如何配置
- CPU密集型：意思是任务需要大量运算，而没有阻塞，cpu一直全速运行，CPU密集型的任务只有在真正多核的CPU上才能得到加速；在单核CPU上，无论有多少个虚拟线程都不可能得到加速
	- CPU密集型任务配置尽量少的线程数量：一般是CPU+1个线程的线程池
- IO密集型：
	- IO密集型任务，线程并不是一直在执行任务，配置尽量多的线程，如：``` CPU核数*2```
	- IO，即有大量的阻塞，在单线程上运行IO密集型任务会导致大量的CPU运算能力浪费等待，所以在IO密集型任务中使用多线程可以大大加速程序运行，即使在单核CPU上，这种加速主要就是利用了被浪费的阻塞时间；；；；所以
		- IO密集型的时候，大部分线程被阻塞，需要配置的线程数：
		参考公式：CPU核数/1-阻塞系数      （阻塞系数一般在0.8-0.9之间）
		比如8核CPU：8/（1-0.9） = 80

### 线程池原理
提交优先级：核心线程>阻塞队列>非核心线程>拒绝策略
执行优先级：

## 死锁编码及其定位

### 原因：
两个或多个线程在执行过程中，因争夺资源而造成的一种互相等待的现象，若无外力干涉，他们将无法推进下去。

线程A持有资源X去获取资源Y，线程B持有资源Y去获取资源X，同时进行，将会发生死锁

### jps定位进程，jstack查看栈信息





## 线程辅助类

### CountDownLatch计数器
```
//调用CountDownLatch的await方法会阻塞当前线程，当计数器变为0的时候，继续执行
	CountDownLatch countDown = new CountDownLatch(5);

        for(int i=0;i<5;i++){
            new Thread(()->{
                countDown.countDown();
                System.out.println(Thread.currentThread().getName() + ":finished");
            }, String.valueOf(i)).start();
        }
        countDown.await();
        System.out.println(Thread.currentThread().getName() + ":all task finished;");
```
### CyclicBarrier  循环屏障

```
//当一组线程都到达指定位置的时候，所有被拦截的线程才会继续运行
//await()调用这个方法后，当前线程阻塞，等待其他线程到达指定节点，都到达后，继续运行；
//CyclicBarrier的第二个参数是回调函数;其他线程会等待这个回调执行完成才继续执行
	CyclicBarrier cyc =  new CyclicBarrier(7, ()->{
            System.out.println("all finished");
        });
        for(int i=0;i<7;i++){
            final int num = i;
            new Thread(()->{
                System.out.println(Thread.currentThread().getName() + "task" + num + "finished");
                try {
                    cyc.await();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }, String.valueOf(i)).start();
        }

```
### Semaphore 信号量 [ˈseməfɔːr]
acquire标识资源抢占；release标识资源释放；
信号灯初始化的参数表示可以抢占的资源，比如下面示例，共有3个资源，只有3个线程可以抢占到，其他线程只能等待这几个线程释放资源后，才能运行

*  用于共享资源的互斥使用
*  控制并发线程数
```
//
		Semaphore sign = new Semaphore(3);
        for(int i=0;i<6;i++){
            new Thread(()->{
                try {
                    sign.acquire();
                    System.out.println(Thread.currentThread().getName() + "获得资源");
                    Thread.sleep(2000);
                    System.out.println(Thread.currentThread().getName() + "处理完成");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }finally{
                    sign.release();
                    System.out.println(Thread.currentThread().getName() + "释放资源");
                }
            }, String.valueOf(i)).start();
        }

```

## 线程通信的几种方式
- volatile 
- Object类的wait() 和 notify() 方法 配合synchronized使用
- 几种线程辅助类：CountDownLatch、CyclicBarrier、Semaphore
- ReentrantLock 结合 Condition
- LockSupport实现线程间的阻塞和唤醒
## 