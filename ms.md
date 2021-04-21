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

### 循环依赖