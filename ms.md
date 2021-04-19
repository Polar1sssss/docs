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

**原理**

每个锁对象拥有一个锁计数器和一个指向持有该锁线程的指针。

当执行monitorenter时，如果目标锁对象的计数器为0，说明该锁没有被其他对象所持有，JVM会将该锁对象的持有线程设置成当前线程，并将其计数器加1。

在目标锁计数器不为0的情况下，如果锁对象的持有线程是当前线程，那么JVM将其计数器加1，否则需要等待，直至持有锁线程释放锁。

当执行monitorexit时，JVM则需要将锁对象的计数器减1。计数器为0代表锁已释放。

#### 显示锁ReentrantLock



