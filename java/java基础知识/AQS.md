# 三入
## 字符串常量池4
java常量池启动的时候会有一个对象："java"
### String.intern()
如果常量池包含这个对象，直接返回常量池中这个对象的引用，如果没有，就将这个字符串放入常量池，并且返回这个变量的引用

### java.misc.Version



## AQS前置认识
### 可重入锁
也叫递归锁，可重复递归调用的锁，在外层使用锁之后，内层仍然可以使用，并且不会发生死锁，这样的锁叫可重入锁，在一个Synchronized修饰的方法或者代码块内部，调用本类的其他Synchronized修饰的方法或者代码块的时候，是永远可以获取到锁的
#### 可重入的说明
每个所对象拥有一个锁计数器和一个指向持有锁线程的指针
当执行monitorenter时，如果目标锁对象的计数器为0，那么说明它没有被其他线程所持有，java虚拟机会将该所对象的持有线程设置为当前线程，并且将其计数器+1
在目标锁对象计数器不为0的时候，如果锁对象的持有线程是当前线程，那么java虚拟机可以将其计数器加1，否则需要等待，直至持有线程释放该锁

当执行monitorexit的时候，java虚拟机将所对象的计数器减一，计数器为0代表锁已被释放

## LockSupport
相当于线程等待唤醒机制（wait/notify）的加强版
park()相当于阻塞线程
unpark()相当于解除阻塞


### 三种线程等待和唤醒的方法
- 使用Object中的wait让线程等待，使用Object中的notify唤醒线程，只在synchronized中有效
	- 不在同步代码块，会报出错误：java.lang.IllegalMonitorStatException;
	- 如果notify在wait之前执行了，wait就会一直阻塞，程序不能退出
- 使用JUC包中的Condition的await让线程等待，使用signal方法唤醒线程，只在lock包含的部分有效
- LockSupport类可以阻塞当前线程以及唤醒指定被阻塞相的线程，底层使用一个permit标识，
	- 调用park方法，设置permit为0.知道别的线程将当前线程的permit设置为1的时候，park方法会被唤醒，然后会将permit再次设置为0并返回
	- 调用unpark方法，会将线程的许可permit设置成1（多次调用， 不会累加，permit还是1），会自动唤醒线程，即之前阻塞中的LockSupport.park()方法会立即返回

**形象的说明：调用park方法，如果有凭证，消耗一个凭证后正常退出，如果没有凭证，就必须阻塞，等待凭证可用；调用unpark方法，他会增加一个凭证，但最多增加一个，累加无效**

### 为什么LockSupport可以先唤醒后阻塞线程？
因为unpark获得了一个凭证之后，调用park方法，就可以直接消费凭证，所以不会阻塞

### 为什么唤醒两次后，调用两次则色，线程结果还是阻塞的？
因为凭证的数量最多是1，连续调用unpark。凭证还是1个，而调用两次park需要消费两个凭证，所以最后是阻塞的

## AQS
AbstractQueuedSynchronizer抽象队列同步器，是用来构建锁或者其他同步器组件的重量级基础框架及整个JUC体系的基石；使用一个state表示锁的占用情况，一个CLH队列来管理被阻塞的线程；实现了出入队、state维护、waitStatus维护等方法，提供了一些模板方法供子类实现；线程阻塞和唤醒依赖于LockSupport
### AQS初步认识
AQS使用一个volatile的int类型的成员变量来标识同步状态，通过内置的FIFO队列（变种的CLH队列，双向的）来完成资源获取的派对工作，将每个要抢占资源的线程封装成一个Node节点来实现锁的分配，通过CAS完成对State值的修改
### 内部结构

### 从ReentrantLock非公平锁认识AQS
Lock接口的实现类，基本都是通过聚合一个队列同步器的子类来完成线程访问控制的
从ReentrantLock中的内部类Sync继承了AQS
ReentrantLock的非公平锁（NofairSync）和公平锁（FairSync）继承了Sync实现自身逻辑
**公平锁直接调用acquire，tryAcquire也是先检查有没有等待的线程，有的话就入队，没有才尝试抢占线程，如果是重入的话就直接加锁**

#### lock
加锁过程可以分为三个阶段：
- 尝试加锁，查看状态是否是0，如果是0，设置当前线程为活跃线程
- 加锁失败，线程进入队列，状态不是期望的0，进入队列
	- acquire以非公平锁实现为例
		- tryAcquire（nofairTryIllegalMonitorStateExceptionAcquire）：再次CAS尝试设置state为1，成功，当前线程设置为活跃线程，返回true；设置失败，判断当前线程是否是活跃线程，如果不是进行下一步（这是一个可重入判断，如果当前线程再次请求获取锁时，会将状态加1，所以状态是可以大于1的）
		- addWaiter：加入队列，采用自旋的方式直至加入成功
			- 如果尾节点是null，加入队列；设置一个空节点作为头节点，将为节点也指向这个节点；如果不是null，将当前节的前指针设置为为节点，CAS设置当前节点为尾节点，成功，则将之前尾节点的后指针设置为当前节点
			- 将当前节点前节点指向头结点，CAS设置为节点为当前节点，设置成功就设置头结点的下一个节点指向当前节点
		- acquireQueued：自旋方式
			- 获取前节点，如果是头节点，再次调用tryRequire尝试抢占锁；抢占成功，设置当前节点为头节点，当前节点的thread为null，前节点引用为null；设置原头头节点后节点引用为null，
			- 抢占失败后
				- shouldParkAfterFailedAcquire：获取前节点的waitStatus，如果是-1，返回true；如果大于0，设置前节点引用为前节点的前节点（这里是为了清除无效节点）， 否则CAS设置前节点的waitStatus为-1
					-waitStatus状态1表示当前节点已经取消调度， 超时或者中断进入；-1：表示后边的节点在等待当前接地那唤醒，后续节点入队会修改前节点Wie这个状态，如果前节点是1，就清除前节点继续向前；-2：表示结点等待在Condition上，当其他线程调用了Condition的signal()方法后，CONDITION状态的结点将从等待队列转移到同步队列中，等待获取同步锁。-3：共享模式下，前节点不仅会唤醒后节点，也会唤醒后节点的后节点；0：新节点入队默认
				- && parkAndCheckInterrupt：使用LockSupport.park()阻塞线程
**1.简单来说：尝试抢占锁；失败调用acquire；2.调用实现的tryAcquire：这时候如果状态是0，尝试抢占锁；如果不是0，进行重入判断； 3. 还没有抢占到锁：进行入队操作：addWaiter；入队成功后；进行acquireQueued，查看前节点是否是头节点，如果是，表示马上轮到自己获取资源，尝试抢占锁；失败或者前节点不是头节点，尝试中断线程；抢占成功或者异常进入cancelAcquire从队列中删除节点，如果node是尾节点，CAS设置尾节点为node的前节点，如果不是尾节点：判断并设置前节点waitStatus为-1，后节点不是空，就把后节点挂接在前节点后；如果前节点失效，就进入unparkSuccessor，初始化当前节点waitStatus=0，遍历后节点，找到一个正常节点进行工作；**
#### unock
调用sync的release方法
- tryRelease尝试释放资源
	- 获取状态减一的值c
	- 如果当前线程不是活跃线程，抛出异常IllegalMonitorStateException
	- 如果c是0，设置当前占用线程是null
	- 设置当前状态为c
	- 如果状态是0，返回true
- 成功（true）
	- 判断头节点是否为null&&头节点的状态是否不为0，如果条件成立
		- unParkSuccessor：
			- 如果头节点waitStatus的状态<0，设置为0
			- 判断下一个节点的状态，如果是null || waitStatus > 0，设置下一个节点为null........，否则使用LockSupport.unPark唤醒线程

**简单来说：release进入后，先tryRelease，尝试释放锁；成功后查看等待队列，设置头节点状态为0，头节点有效；从队尾开始遍历获取一个有效节点解除阻塞**

#### 为什么出队从队尾遍历
因为入队的时候，使用CAS设置了prev指向前节点，而前节点的next指针设置和这一步是不一个原子操作，从头部遍历可能会漏掉新入队的元素；从尾部就没问题了
### CLH

acquire调用tryAcquire
release调用tryRelease
tryAcquire和tryRelease需要实现着自己实现，模板方法模式

内部维护一个原子整型state和CLH双向队列
state=0表示没有线程占用这把锁

### 从ReentrantReadWriteLock非公平锁认识AQS
使用state的高16位和低16位分别维护读锁（共享锁）和写锁（独占锁）
HoldCounter的作用就是当前线程持有共享锁的数量，这个数量必须要与线程绑定在一起，否则操作其他线程锁就会抛出异常

#### 写加锁
- lock
- tryAcquire
    - tryAcquireShared进行重入判断；尝试加锁
- 后续同ReentrantLock
#### 写解锁
- unlock
- tryRelease：状态-1；成功解锁；不为0，设置状态

#### 读加锁
- acquireShared调用
	- tryAcquireShared：写锁被其他线程（不是当前）占用调用
		- doAcquireShared：读锁入队操作，成功后，查看上一个节点是否是头节点；如果是头节点，再次尝试获取锁；如果不是头节点，阻塞当前线程；
	- 尝试获取锁：状态=0：直接占用；是当前线程：重入；
	- 读锁之前已经被共享了，当前线程不是获取读锁的首个线程
		- 当前线程不是缓存的线程；获取当前线程对应的计数器；否则如果之前获取的线程计数是0，加入到readHolds中
    	- 缓存线程计数+1
    - CAS设置状态失败，进入fullTryAcquireShared
    	- 自旋获取读锁；直至获取到锁，或者进入缓存


### 读写锁锁降级：
当数据变动时，所有所有readwrite()方法的线程都能感知到变化，但是只有一个线程能够获取到写锁，其他线程会被阻塞在读锁和写锁的lock()方法上。当前线程获取写锁完成数据准备之后，再获取读锁，随后释放写锁，完成锁降级

**锁降级是指线程先持有写锁，再获取到读锁，随后释放（先前拥有的）写锁的过程；**

可能存在一个事务线程不希望自己的操作被别的线程中断，而这个事务操作可能分成多部分操作更新不同的数据（或表）甚至非常耗时。如果长时间用写锁独占，显然对于某些高响应的应用是不允许的，所以在完成部分写操作后，退而使用读锁降级，来允许响应其他进程的读操作。只有当全部事务完成后才真正释放锁。
```
private Map<String, Object> map = new HashMap<>();

    private ReadWriteLock rwl = new ReentrantReadWriteLock();

    private Lock r = rwl.readLock();
    private Lock w = rwl.writeLock();

    private volatile boolean isUpdate;

    public void readWrite() {
        isUpdate = true;
        w.lock();
        map.put("xxx", "xxx");
        r.lock();
        w.unlock();
        r.unlock();
    }

```