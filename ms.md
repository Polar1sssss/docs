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

------

第一层缓存singletonObjects存放已经初始化好的Bean；第二层缓存earlySingletonObjects存放实例化但未初始化的Bean；第三层缓存singletonFactories存放的是FactoryBean，如果某个类实现了FactoryBean，那么依赖注入的不是该类，而是该类对应的Bean。

------

A/B两个互相依赖的对象在三级缓存中迁移说明？

1. A创建过程中需要用到B，于是A将自己放到三级缓存里面，去实例化B
2. B实例化的时候需要A，于是B先查一级缓存，没有，再查二级缓存，还没有，再查三级缓存，找到A并把三级缓存中的A放到二级缓存里面，并删除三级缓存里面的A
3. B顺利初始化完毕，将自己放到一级缓存中（此时B里面的A依然是创建中状态），然后回来接着创建A，此时B已经创建结束，直接从一级缓存中拿到B，完成A的创建并把A放到一级缓存中

------

Spring解决循环依赖依靠的是Bean“中间态”的概念，即已经实例化但是还未初始化的状态。实例化过程是通过构造器实现的，如果A还未创建好则不能提前暴露，所以构造器注入无法解决循环依赖问题。

------

![ms-007](https://cdn.jsdelivr.net/gh/Polar1sssss/FigureBed/img/ms-007.png)

Spring解决循环依赖过程：

1. 调用doGetBean()方法，想要获取beanA，于是调用getSingleton()方法从缓存中查找beanA
2. 在getSingleton()方法中，从【一级缓存】中查找，没有，返回null
3. doGetBean()方法中获取到的beanA为null，于是走对应的处理逻辑，调用getSingleton()的重载方法（参数为ObjectFactory的)
4. 在getSingleton()方法中，先将beanA_name添加到一个集合中，用于标记该bean正在创建中。然后回调匿名内部类的creatBean方法
5. 进入AbstractAutowireCapableBeanFactory#doCreateBean，先利用反射创建出beanA的实例，然后判断：是否为单例、是否允许提前暴露引用(对于单例一般为true)、是否正在创建中（即是否在第四步的集合中）。判断为true则将beanA添加到【三级缓存】中
6. 对beanA进行属性填充（populateBean），此时检测到beanA依赖于beanB，于是开始查找beanB
7. 调用doGetBean()方法，和上面beanA的过程一样，到缓存中查找beanB，没有则先在【三级缓存】中创建，然后给beanB填充属性
8. 此时 beanB依赖于beanA，调用getSingleton()获取beanA，依次从一级、二级、三级缓存中找，此时从【三级缓存】中获取到beanA的创建工厂，通过创建工厂获取到singletonObject，此时这个singletonObject指向的就是上面在doCreateBean()方法中实例化的beanA
9. 将beanA从【三级缓存】移动到【二级缓存】，beanB就获取到了beanA的依赖，beanB顺利完成实例化，从【三级缓存】直接移动到【一级缓存】
10. 随后beanA继续属性填充工作，从【一级缓存】中获取到beanB，beanA完成初始化，回到getsingleton()方法中继续向下执行，将beanA从二级缓存移动到一级缓存中

## Redis

string  list  hash  set  zset  bitmap  hyperloglog  geo

命令关键字不区分大小写，key区分大小写

### 五大数据类型的落地应用

#### String

- 设置获取：set key value / get key

- 批量设置获取：mset key value[key1 value1...] / mget key [key1]

- 递增：incr key / incrby key increment

- 递减：decr key / decrby key increment

- 获取字符串长度：strlen key 

- 分布式锁：

  ​	setnx key value

  ​	set key value [EX seconds] [PX milliseconds] [NX|XX]

- 参数说明：

  ​	EX -- key在多少秒后过期    

  ​	PX -- key在多少毫秒之后过期    

  ​	NX -- 当key不存在的时候创建key，效果等同于setnx    

  ​	XX -- 当key存在时候，覆盖key 

- 应用场景：商品编号、订单号、点赞、踩：incr item:001


#### Hash

Map<String, Map<Object, Object>>

- 设置获取：hset key field value / hset key field

- 批量设置获取：hmset key field value [field1 value1...] / hmset key field [field1]

- 获取所有字段值：hgetall key

- 获取key中字段的数量：hlen key

- 删除key：hdel key

- 应用场景：

  ​	购物车早期，中小厂可用

  ​	新增商品：hset shopcar:1024 11111 1

  ​	新增商品：hset shopcar:1024 22222 1

  ​	增加商品商量：hincrby shopcar:1024 22222 1

  ​	全部选择：hgetall shopcar:1024

#### List

- 向列表左边/右边添加元素：lpush/rpush key value [value...]
- 查看列表：lrange key start stop
- 获取列表中元素个数：llen key
- 应用场景：订阅的公众号推送文章

#### Set

- 添加元素：sadd key member[member1...]
- 删除元素：srem key member[member1]
- 获取集合中所有元素：smembers key
- 判断元素是否在集合中：sismember key member
- 获取集合中元素个数：scard key
- 从集合中随机弹出一个元素，元素不删除：srandmember key [弹出元素个数]
- 从集合中随机弹出一个元素，元素删除：spop key [弹出元素个数]
- 集合运算：
  - 差集：sdiff key [key1]
  - 交集：sinter key [key1]
  - 并集：sunion key [key1]
- 应用场景：
  - 微信抽奖小程序：参与抽奖（sadd）、显示参与人数（scard）、抽奖（srandmember、spop）
  - 微信朋友圈点赞：点赞（sadd 朋友圈id 点赞用户id1 点赞用户id2）、取消点赞（srem 朋友圈id 点赞用户id1）、展现所有点赞用户（smember 朋友圈id）、点赞数（scard 朋友圈id）
  - 微博好友关注社交关系：共同关注的人（sinter k1 k2）、我关注的人也关注他（sismember k1 v、sismember k2 v 都返回true）
  - QQ推送可能认识的人：A认识B不认识的推送给B（sdiff A B）

#### Zset

向有序集合中加入一个元素和该元素的分数。

- 添加元素：zadd key score member [score1 member1...]
- 按照分数从小到大顺序返回索引在start到stop之间的元素：zrange key start stop [WITHSCORES]
- 获取元素的分数：zscore key member
- 删除元素：zrem key member [member1]
- 获取指定分数范围的元素：zrangebyscore key min max [WITHSCORES] [LIMIT offset count]
- 增加某个元素的分数：zincrby key increment member
- 获取集合中元素的数量：zcard key
- 获取指定分数范围内元素个数：zcount key min max
- 按照排名范围删除元素：zremrangebyrank key start stop
- 获取元素的排名：
  - 从小到大：zrank key member
  - 从大到小：zrevrank key member
- 应用场景：
  - 根据商品销售量对商品排序展示
  - 抖音热搜

### 分布式锁

```java
/**
 * 1、多线程情况下出现安全问题：加单机锁synchronized或者Lock
 * 2、使用nginx组成分布式架构，单机锁无用：使用redis的setNx命令加分布式锁
 * 3、加分布式锁出现异常，解锁操作无法进行：解锁操作放到finally代码块中
 * 4、微服务服务器宕机，分布式锁的key永远无法删除：对分布式锁设置过期时间
 * 5、加锁操作和设置过期时间代码不在一行，不具备原子性：使用setIfAbsent(String key, String value, 	
 * long timeout, TimeUnit unit)
 * 6、过期时间内业务没有处理完成，会出现删除别人的锁的现象（加锁解锁不是同一个客户端）：解锁之前判断value是 	* 否是加锁时的value
 * 7、分布式锁的判断和删除非原子操作：LUA脚本、事务
 * 8.1、确保RedisLock过期时间大于业务执行之间：分布式锁续期
 * 8.2、AP，redis主从异步复制导致数据丢失（主机没来得及同步锁就挂掉了，从机上不具备锁信息）
 * 8问题的解决方案：RedLock，在java中具体实现为Redisson
 * 伪代码，##和**代表配套使用的部分
 */
@RestController
public class TestDistributeLock {
    public static final LOCK_NAME = "abc";
    @Autowired
    private StringRedisTemplate template;
    @Autowired
    private Redisson redisson;
    
    @GetMapping("/buyGoods")
    public String buyGoods() {
       	String value = UUID.randomUUID().toString + Thread.currentThread.getName();
        
        // ##redisson方式加锁
        RLock redissonLock = redisson.getLock(LOCK_NAME);
        redissonLock.lock();
        
        try {
            // 加分布式锁的时候设置过期时间，保证原子性操作
            Boolean flag = template.opsForValue().setIfAbsent(LOCK_NAME, value, 10L, TimeUnit.SECONDS);
            if (!flag) {
                return "抢锁失败";
            }
            String result = template.opsForValue().get("goods:001");
            int goodsNum = result == null ? 0 : Integer.parseInt(result);

            if (goodsNum > 0) {
                int realNum = goodsNum - 1;
                template.opsForValue().set("goods:001", String.valueOf(realNum));
                return "成功买到商品，库存剩余" + realNum;
            } 
            return "商品售罄~";
        } finally {
            // 事务
            while (true) {
                template.watch(LOCK_NAME);
                // 删除key时判断，是否是加锁的客户端在执行操作
                if (template.opsForValue().get(key).equals(value)) {
                    template.setEnableTransactionSupport(true);
                    template.multi();
                    template.delete(LOCK_NAME);
                    List<Object> list = template.exec();
                    // 执行失败重试
                    if (list == null) {
                        continue;
                    }
                }
                template.unwatch();
                break;
            }
            
            // **LUA
            Jedis jedis = RedisUtils.getJedis();
            String script = "if redis.call('get', KEYS[1]) == ARGV[1] " + 
                "then " +
                " return redis.call('del', KEYS[1]) " +
                "else " + 
                " return 0 " + 
                "END";
            try {
                Object o = jedis.eval(script, Collections.singletonList(REDIS_LOCK), Collections.singletonList(value));
                if ("1".equals(o)) {
                    System.out.println("删除redis lock成功")；
                } else {
                    System.out.println("删除redis lock失败")；
                }
            } finally {
                // **Jedis解锁
                if (null != jedis) {
                    jedis.close();
                }
                
                // ##Redisson方式解锁操作
                if (redissonLock.isLocked() && redissonLock.isHeldByCurrentThread()) {
                    redissonLock.unlock();
                }
            }
        }
    }
}
```



### 内存淘汰策略

redis默认内存大小：如果不设置最大内存大小或者最大内存大小为0，在64位操作系统下不限制内存大小，32位操作系统下最多使用3GB内存。默认单位是字节，配置时要注意单位转换。

生产上内存大小最好设置成最大物理内存的四分之三。

设置内存大小方式：

- 修改配置文件  maxmemory 104857600

- 通过命令修改

  ```shell
  >> config get maxmemory
  >> config set maxmemory 1
  ```

查看redis内存情况：info memory

过期键删除策略：定时删除（键过期立即删除）、惰性删除（使用键时判断是否过期，过期则删除）、定期删除（抽样删除过期键）

内存淘汰策略：noevication（默认）、allkeys-lru、volatile-lru、allkeys-random、volatile-random、allkeys-lfu、volatile-lfu、volatile-ttl

### lru算法

lru算法思想：查找快、插入快、删除快、且要有先后排序

lru算法核心：哈希链表LinkedHashMap，本质是HashMap+DoubleLinkedList，时间复杂度是O(1)

案例一：

```java
/**
 * 继承LinkedHashMap
 */
public class LRUCacheDemo<K, V> extends LinkedHashMap<K, V> {
    private int capcity;
    
    public LRUCacheDemo(int capcity) {
        // 参数说明：容量，负载因子，访问顺序：true--按照访问频率排序；false--按照插入顺序排序
        super(capcity, 0.75F, true);
        this.capcity = capcity;
    }
    
    // 重写父类 移除最少使用元素 的方法
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return super.size() > capcity;
    }
    
    public static void main() {
        LRUCacheDemo lru = new LRUCacheDemo();
        lru.put(1, "a");
        lru.put(2, "b");
        lru.put(3, "c");
        System.out.println(lru.keySet()); // [1,2,3] [1,2,3]
        
        lru.put(4, "d");
        System.out.println(lru.keySet()); // [2,3,4] [2,3,4]
        
        lru.put(3, "c");
        System.out.println(lru.keySet()); // [2,4,3] [2,3,4]
        lru.put(3, "c");
        System.out.println(lru.keySet()); // [2,4,3] [2,3,4]
        lru.put(5, "x");
        System.out.println(lru.keySet()); // [4,3,5] [3,4,5]
    }
}
```

案例二：

```java
/**
 * 手写LRU
 */
public class LRUCache {
    
    // map负责查找，构建一个虚拟的双向链表，它里面安装的就是一个Node节点，作为数据载体
    // 1.构造一个Node节点作为数据载体
    class Node<K, V> {
        K key;
        V value;
        Node<K, V> prev;
        Node<K, V> next;
        
       	public Node() {
            this.prev = this.next = null;
        }
        
        public Node(K key, V value) {
            this.key = key;
            this.value = value;
            this.prev = this.next = null;
        }
    }
    
    // 2.构造一个双向链表，用于存放Node节点
    class DoubleLinkedList<K, V> {
        Node<K, V> head;
        Node<K, V> tail;
        
        public DoubleLinkedList() {
            head = new Node<>();
            tail = new Node<>();
            head.next = tail;
            tail.prev = head;
        }
        
        // 添加节点
        public void addHead(Node<K, V> node) {
            // 当前节点后继指针指向头节点后继节点（尾节点）
            node.next = head.next;
            // 尾节点前驱指针指向当前节点
            head.next.prev = node;
            // 头节点后继指针指向当前节点
            head.next = node;
            // 当前节点前驱指针指向头节点
            node.prev = head;
        }
        
        // 删除节点
        public void removeNode(Node<K, V> node) {
            node.next.prev = node.prev;
            node.prev.next = node.next;
            // 当前节点的指针全部指向空，相当于出队
            node.prev = null;
            node.next = null;
        }
        
        // 获得最后一个节点
        public Node getLast() {
            return tail.prev;
        }
    }
    
    // 容量大小
    private int cacheSize;
    Map<Integer, Node<Integer, Integer>> map;
    DoubleLinkedList<Integer, Integer> doubleLinkedList;
    
    public LRUCache(int cacheSize) {
        this.cacheSize= cacheSize；
        map = new HashMap<>();
        doubleLinkedList = new DoubleLinkedList<>();
    }
    
    
    public int get(int key) {
    	if (!map.containsKey(key)) {
            reutrn -1;
        }
        
        Node<Integer, Integer> node = map.get(key);
        // 频繁使用（读和写）的现从链表中删除，再加到队头
        doubleLinkedList.removeNode(node);
        doubleLinkedList.addHead(node);
    }
    
    // 更新和新增
    public void put(int key, int value) {
        // 更新节点
        if (map.contains(key)) {
            Node<Integer, Integer> node = map.get(key);
            node.value = value;
            map.put(key, node);
            
            doubleLinkedList.removeNode(node);
       		doubleLinkedList.addHead(node);
        } else {
            // 容量达到最大值
            if (map.size() == cacheSize) {
                Node<Integer, Integer> lastNode = doubleLinkedList.getLast();
                map.remove(lastNode.key);
                doubleLinkedList.removeNode(lastNode);
            }
                    
            // 新增节点
            Node<Integer, Integer> newNode = new Node<>(key, value)
            map.put(key, newNode);
            doubleLinkedList.addHead(newNode);
        }
    }
    
    public static void main(String[] args) {
        // 测试同上
    }
}
```

![ms-010](https://cdn.jsdelivr.net/gh/Polar1sssss/FigureBed/img/ms-010.png)