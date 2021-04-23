## String与String.intern()

```java
public class RuntimeConstantPoolOOM {
    public static void main(String[] args) {
        String str1 = new StringBuilder("计算机").append("软件").toString();
        System.out.println(str1.intern() == str1); // true
        
        String str2 = new StringBuilder("ja").append("va").toString();
        System.out.println(str2.intern() == str2); // false
    }
}
```

这段代码在JDK 6中运行，会得到两个false，而在JDK 7中运行，会得到一个true和一个false。产 生差异的原因是，在JDK 6中，intern()方法会把首次遇到的字符串实例复制到永久代的字符串常量池 中存储，返回的也是永久代里面这个字符串实例的引用，而由StringBuilder创建的字符串对象实例在 Java堆上，所以必然不可能是同一个引用，结果将返回false。

而JDK 7（以及部分其他虚拟机，例如JRockit）的intern()方法实现就不需要再拷贝字符串的实例到永久代了，既然字符串常量池已经移到Java堆中，那只需要在常量池里记录一下首次出现的实例引用即可，因此intern()返回的引用和由StringBuilder创建的那个字符串实例就是同一个。而对str2比较返回false，这是因为“java” 这个字符串在执行StringBuilder.toString()之前就已经出现过了（**在加载sun.misc.Version这个类的时候进入常量池的**），字符串常量池中已经有它的引用，不符合intern()方法要求“首次遇到”的原则，“计算机软件”这个字符串则是首次 出现的，因此结果返回true。

## AQS

### 可重入锁

> **可重复递归调用的锁，在外层获得锁之后，在内层无需等待锁的释放，可以直接使用，并且不会发生死锁**

#### 隐式锁synchronized

- 原理

  每个锁对象拥有一个锁计数器和一个指向持有该锁线程的指针。

  当执行monitorenter时，如果目标锁对象的计数器为0，说明该锁没有被其他对象所持有，JVM会将该锁对象的持有线程设置成当前线程，并将其计数器加1。

  在目标锁计数器不为0的情况下，如果锁对象的持有线程是当前线程，那么JVM将其计数器加1，否则需要等待，直至持有锁线程释放锁。

  当执行monitorexit时，JVM则需要将锁对象的计数器减1。计数器为0代表锁已释放。

#### 显示锁ReentrantLock

### LockSupport

#### 3种线程等待唤醒机制

- Object中的wait() / notify() / notifyAll()
  - 不能脱离synchronized代码块使用，否则会抛出IllegalMonitorStateException异常
  - 先wait后notify、notifyAll，等待中的线程才能被唤醒，顺序不能改变
- Condition的await() / siginal() / siginalAll()
  - 必须配合lock()方法使用，否则抛出IllegalMonitorStateException异常
  - 等待唤醒调用顺序不能改变
- LockSupport提供park()和unpark()方法实现阻塞和唤醒线程的过程
- LockSupport和每一个使用它的线程之间有一个许可（permit）进行关联，permit有0和1两种状态，默认为0，即无许可证状态
- 调用一次unpark方法，permit加1变成1。每次调用park方法都会检查许可证状态，如果为1，则消耗掉permit（1 -> 0）并立刻返回；如果为0，则进入阻塞状态。permit最多只有一个，重复调用unpark也不会累积permit。

### AQS源码分析

AQS：用来构建锁或其他同步器组件的重量级基础框架及整个JUC体系的基石，通过内置的FIFO队列来完成资源获取线程的排队工作，并通过一个int类型变量表示持有锁的状态。如果共享资源被占用，就需要一定的阻塞等待唤醒机制来保证锁的分配。主要通过CLH队列的变体实现，将暂时获取不到锁的线程加入到队列中，这个队列就是AQS抽象的表现。它将请求共享资源的线程封装成队列的节点（Node），通过CAS自旋以及LockSupport.park()的方式，维护state变量的状态，使并发达到同步的控制效果。

CLH队列：Craig、Landin and Hagersten队列，是一个单向链表，AQS中的队列是CLH变体的虚拟双向队列，虚拟的双向队列即不存在队列实例，仅存在节点之间的关联关系。

![](https://cdn.jsdelivr.net/gh/Polar1sssss/FigureBed/img/ms-001.png)

AQS相关类：ReentrantLock、CountDownLatch、CyclicBarrier、ReentrantReadWriteLock、Semaphore

![image-20210420101312775](https://cdn.jsdelivr.net/gh/Polar1sssss/FigureBed/img/ms-002.png)

#### 通过ReentrantLock类进行源码分析

以非公平锁为例：

<font color = "red">lock()</font>

```java
static final class NonfairSync extends Sync {
    
	private static final long serialVersionUID = 7316153563782823691L;
    final void lock() {
        // 首先进行一次cas抢锁操作，成功则获取锁
        if (compareAndSetState(0, 1))
            // 第一个线程抢占
            setExclusiveOwnerThread(Thread.currentThread());
        else
            // 第二个及后续线程抢占
            acquire(1);
    }

    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }
    
}
     
```

<font color = "red">acquire()</font>

```java
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    implements java.io.Serializable {	
    
	public final void acquire(int arg) {
        if (!tryAcquire(arg) && // 交给子类实现
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
    
}
```

<font color = "red">tryAcquire(arg)</font>

```java
abstract static class Sync extends AbstractQueuedSynchronizer {
    
    private static final long serialVersionUID = -5179523762034025860L;
    final boolean nonfairTryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {  // 如果刚好锁被释放，那么再进行一次cas抢锁操作
            if (compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) { // 可重入锁
            int nextc = c + acquires;
            if (nextc < 0) // overflow
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
    
}
```

<font color = "red">addWaiter(Node.EXCLUSIVE)</font>

```java
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    implements java.io.Serializable {    
    
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        enq(node);
        return node;
    }

    private Node enq(final Node node) {
        for (;;) {
            Node t = tail;

            if (t == null) { // Must initialize，之前没有节点入过队
                // 双向链表中，第一个节点为虚节点（哨兵节点），不存储任何信息，只是占位。
                // 真正第一个有数据的节点从第二位开始
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
       
}
```

<font color = "red">acquireQueued(addWaiter(Node.EXCLUSIVE), arg)</font>

```java
// ****AbstractQueuedSynchronizer****
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        // 自旋
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                // 抢锁成功，将当前持有锁的线程节点设置为新的哨兵节点（节点中线程指向null）
                // 清除之前的哨兵节点
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}

private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    // 获取前驱节点的状态
    int ws = pred.waitStatus;
    // 如果是SIGNAL状态，即等待被占用资源释放，直接返回true
    // 准备继续调用parkAndCheckInterrupt方法
    if (ws == Node.SIGNAL)
        return true;
    // ws大于0证明是CANNELED状态
    if (ws > 0) {
        // 循环判断前驱节点的前驱节点状态是否也是CANNELED，忽略该状态的节点
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        // 将当前节点的前驱节点状态设置为SIGNAL，用于后续唤醒操作
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}

private final boolean parkAndCheckInterrupt() {
    // 运行到这线程被挂起，才算真正入队
    LockSupport.park(this);
    // 根据park方法API描述，程序在以下三种情况会继续向下执行
    // 1.被unpark
    // 2.被中断（interrupt）
    // 3.其他不合逻辑的返回
   	// interrupted：检查当前线程是否被中断，返回布尔值并清除中断状态
    // isInterrupted：只测试此线程是否被中断 ，不清除中断状态
    return Thread.interrupted();
}
```

------

<font color="green">unlock()</font>

```java
// ****ReentrantLock****
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}

protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {
        free = true;
        // 将独占锁拥有者设置为空
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```

<font color="green">unparkSuccessor()</font>

```java
// ****AbstractQueuedSynchronizer****
private void unparkSuccessor(Node node) {
   
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        // 唤醒哨兵节点的下一个节点
        LockSupport.unpark(s.thread);
}
```

## Spring

### AOP的顺序

#### 通知类型

@Before：前置通知，目标方法之前执行

@After：后置通知，目标方法之后执行（始终执行）

@AfterReturning：返回通知，执行方法结束前执行（异常不执行）

@AfterThrowing：异常通知，出现异常时执行

@Around：环绕通知，环绕目标方法执行

#### SpirngBoot1和SpringBoot2对aop通知执行顺序的影响

Spring4.x

- 正常执行：@Around > @Before > 方法打印 > @Around > @After > @AfterReturning
- 抛出异常：@Around > @Before > @After > @AfterThrowing > 异常信息

Spring5.x

- 正常执行：@Around > @Before > 方法打印> @AfterReturning > @After > @Around 
- 抛出异常：@Around > @Before > @AfterThrowing > @After > 异常信息

### 循环依赖

多个bean之间相互依赖，形成了一个闭环。Spring循环依赖抛出BeanCurrentlyInCreationException。

只要注入方式是setter且是单例的，就不会有循环依赖问题。

#### Spring三级缓存

只有单例的bean会通过三级缓存提前暴露来解决循环依赖的问题，而非单例的bean，每次从容器中获取的都是一个新的对象，都会重新创建，所以非单例的bean是没有缓存的，不会将其放到三级缓存中。

第一级（单例池）：singletonObjects，存放已经经历了完整生命周期的bean对象。

第二级：earlySingletonObjects，存放早期暴露出来的Bean对象，Bean的生命周期未结束（属性还未填充）。

第三级：Map<String,  ObjectFactory<?>> singletonFactories，存放可以生成Bean的工厂。

![ms-004](https://cdn.jsdelivr.net/gh/Polar1sssss/FigureBed/img/ms-004.png)

#### 源码解析

![ms-003](https://cdn.jsdelivr.net/gh/Polar1sssss/FigureBed/img/ms-003.png)

![ms-005](https://cdn.jsdelivr.net/gh/Polar1sssss/FigureBed/img/ms-005.png)

实例化：内存中申请一块空间

初始化：属性填充，完成属性的各种赋值

第一层缓存singletonObjects存放已经初始化好的Bean；第二层缓存earlySingletonObjects存放实例化但未初始化的Bean；第三层缓存singletonFactories存放的是FactoryBean，如果某个类实现了FactoryBean，那么依赖注入的不是该类，而是该类对应的Bean。



A/B两个互相依赖的对象在三级缓存中迁移说明？

1. A创建过程中需要用到B，于是A将自己放到三级缓存里面，去实例化B
2. B实例化的时候需要A，于是B先查一级缓存，没有，再查二级缓存，还没有，再查三级缓存，找到A并把三级缓存中的A放到二级缓存里面，并删除三级缓存里面的A
3. B顺利初始化完毕，将自己放到一级缓存中（此时B里面的A依然是创建中状态），然后回来接着创建A，此时B已经创建结束，直接从一级缓存中拿到B，完成A的创建并把A放到一级缓存中



Spring解决循环依赖依靠的是Bean“中间态”的概念，即已经实例化但是还未初始化的状态。实例化过程是通过构造器实现的，如果A还未创建好则不能提前暴露，所以构造器注入无法解决循环依赖问题。



Spring解决循环依赖过程：

1. 调用doGetBean()方法，想要获取beanA，于是调用getSingleton()方法从缓存中查找beanA
2. 在getSingleton()方法中，从一级缓存中查找，没有，返回null
3. doGetBean()方法中获取到的beanA为null，于是走对应的处理逻辑，调用getSingleton()的重载方法（参数为ObjectFactory的)
4. 在getSingleton()方法中，先将beanA_name添加到一个集合中，用于标记该bean正在创建中。然后回调匿名内部类的creatBean方法
5. 进入AbstractAutowireCapableBeanFactory#doCreateBean，先利用反射创建出beanA的实例，然后判断：是否为单例、是否允许提前暴露引用(对于单例一般为true)、是否正在创建中（即是否在第四步的集合中）。判断为true则将beanA添加到【三级缓存】中
6. 对beanA进行属性填充（populateBean），此时检测到beanA依赖于beanB，于是开始查找beanB
7. 调用doGetBean()方法，和上面beanA的过程一样，到缓存中查找beanB，没有则创建，然后给beanB填充属性
8. 此时 beanB依赖于beanA，调用getSingleton()获取beanA，依次从一级、二级、三级缓存中找，此时从三级缓存中获取到beanA的创建工厂，通过创建工厂获取到singletonObject，此时这个singletonObject指向的就是上面在doCreateBean()方法中实例化的beanA
9. 这样beanB就获取到了beanA的依赖，于是beanB顺利完成实例化，并将beanA从三级缓存移动到二级缓存中
10. 随后beanA继续他的属性填充工作，此时也获取到了beanB，beanA也随之完成了创建，回到getsingleton()方法中继续向下执行，将beanA从二级缓存移动到一级缓存中

## Redis

string  list  hash  set  zset  bitmap  hyperloglog  geo

命令关键字不区分大小写，key区分大小写

### 五大数据类型的落地应用

#### String

设置获取：set key value / get key

批量设置获取：mset key value[key1 value1...] / mget key [key1]

递增：incr key / incrby key increment

递减：decr key / decrby key increment

获取字符串长度：strlen key 

分布式锁：

​	setnx key value

​	set key value [EX seconds] [PX milliseconds] [NX|XX]

说明：

​	EX -- key在多少秒后过期    

​	PX -- key在多少毫秒之后过期    

​	NX -- 当key不存在的时候创建key，效果等同于setnx    

​	XX -- 当key存在时候，覆盖key 

应用场景：商品编号、订单号、点赞、踩：incr item:001

#### Hash

Map<String, Map<Object, Object>>

设置获取：hset key field value / hset key field

批量设置获取：hmset key field value [field1 value1...] / hmset key field [field1]

获取所有字段值：hgetall key

获取key中字段的数量：hlen key

删除key：hdel key

应用场景：

​	购物车早期，中小厂可用

​	新增商品：hset shopcar:1024 11111 1

​	新增商品：hset shopcar:1024 22222 1

​	增加商品商量：hincrby shopcar:1024 22222 1

​	全部选择：hgetall shopcar:1024

#### List



#### Set

#### Zset









