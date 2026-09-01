---
title: Java线程安全
date: 2026-08-15 08:00:00 +0800
categories: [微服务, 高并发IO]
tags: [后端, 微服务, 高并发IO, 线程安全]
music-id: 864711417
---

## **序言**

### **什么是线程安全？**

当多个线程并发访问某个`Java`对象（`Object`）时，无论系统如何调度这些线程，也不论这些线程如何交替操作，这个对象都能表现出一致的、正确的行为，那么对这个对象的操作是线程安全的。如果这个对象表现出不一致的、错误的行为，那么对这个对象的操作不是线程安全的，发生了线程的安全问题。

### **自增运算不是线程安全的**

看上去，对一个整数进行自增运算（`++`），只有一个完整的操作，不可分割。实际上，一个自增运算符是一个复合操作，至少包括三个`JVM`指令：内存取值、寄存器增加`1`、存值到内存。这三个指令在`JVM`内部是独立进行的，中间完全可能会出现多个线程并发进行。

举个例子，`10`个线程并行运行，对一个共享数据进行自增运算，每个线程自增运算`1000`次。具体的代码如下，

```java
public class NotSafePlus
{
    private Integer amount = 0;
    //自增
    public void selfPlus()
    {
        amount++;
    }
    public Integer getAmount()
    {
        return amount;
    }
}
```

在`amount=100`时，假设有三个线程同一时间读取`amount`值，读到的都是`100`，增加`1`后结果为`101`，三个线程都将结果存入到`amount`的内存，`amount`的结果是`101`，而不是`103`。

内存取值、寄存器增加`1`、存值到内存，这三个`JVM`指令是不可以再分的，它们都具备原子性，是线程安全的，是原子操作。但是，两个或者两个以上的原子操作合在一起进行操作就不再具备原子性。比如先读后写，就有可能在读之后，其实这个变量被修改了，就出现了数据不一致的情况。

```java
public class PlusTest
{
    final int MAX_TREAD = 10;
    final int MAX_TURN = 1000;

    /**
     * 测试用例：测试不安全的累加器
     */
    public void testNotSafePlus() throws InterruptedException
    {
        //倒数门，需要倒数MAX_TREAD次
        CountDownLatch latch = new CountDownLatch(MAX_TREAD);
        NotSafePlus counter = new NotSafePlus();
        Runnable runnable = () ->
        {
            for (int i = 0; i < MAX_TURN; i++)
            {
                counter.selfPlus();
            }
            latch.countDown();      // 倒数门减少一次
        };
        for (int i = 0; i < MAX_TREAD; i++)
        {
            new Thread(runnable).start();
        }
        latch.await();              // 等待倒数门的次数减少到0，所有的线程执行完成
        log.info("理论结果： " + MAX_TURN * MAX_TREAD);
        log.info("实际结果： " + counter.getAmount());
        log.info("差距是： " + (MAX_TURN * MAX_TREAD - counter.getAmount()));
    }
}
```

运行程序，输出的结果如下，

```
理论结果: 10000
实际结果: 7006
差距是: 2994
```

通过结果可以看出，总计自增`10000`次，结果少了`2994`次，差距在`30%`左右。这只是一次结果，每一次运行，差距都是不同的。

从结果可以看出，对`NotSafePlus`的`amount`成员的`++`运算在多线程并发执行场景下出现了不一致的、错误的行为，自增运算符`++`不是线程安全的。

### **临界区资源与临界区代码段**

`Java`工程师在进行代码开发时，常常倾向于认为代码会以线性的、串行的方式执行，容易忽视多个线程并行执行，从而导致意想不到的结果。

前面的线程安全实验展示了在多个线程操作相同资源（如变量、数组或者对象）时就可能出现线程安全问题。一般来说，只在多个线程对这个资源进行写操作的时候才会出现问题，如果是简单的读操作，不改变资源的话，显然是不会出现问题的。

临界区资源表示一种可以被多个线程使用的公共资源或共享数据，但是每一次只能有一个线程使用它。一旦临界区资源被占用，想使用该资源的其他线程则必须等待。

在并发情况下，临界区资源是受保护的对象。临界区代码段（`Critical Section`）是每个线程中访问临界资源的那段代码，多个线程必须互斥地对临界区资源进行访问。线程进入临界区代码段之前，必须在进入区申请资源，申请成功之后进行临界区代码段，执行完成之后释放资源。

竞态条件（`Race Conditions`）可能是由于在访问临界区代码段时没有互斥地访问而导致的特殊情况。如果多个线程在临界区代码段的并发执行结果可能因为代码的执行顺序不同而出现不同的结果，我们就说这时在临界区出现了竞态条件问题。

前面的线程安全实验代码中，`amount`为临界区资源，`selfPlus()`可以理解为临界区代码段，具体如下，

```java
public class NotSafePlus
{
    private Integer amount = 0; // 临界区资源
    // 临界区代码段
    public void selfPlus()
    {
        amount++;
    }
}
```

当多个线程访问临界区`selfPlus()`方法时，就会出现竞态条件的问题。更标准地说，当两个或多个线程竞争同一个资源时，对资源的访问顺序就变得非常关键。

为了避免竞态条件的问题，我们必须保证临界区代码段操作必须具备排他性。这就意味着当一个线程进入`Critical Section`执行时，其他线程不能进入临界区代码段执行。

## **保证线程安全的方法有哪些？**

线程安全的核心是，解决多线程同时访问共享资源时出现的原子性、可见性、有序性问题。这里把常用的方法归纳为`5`大类，从简单到复杂依次如下，

### **无状态/不可变对象**

- 原理：没有共享可变状态，天然线程安全
- 实现：
    - 使用 `final` 修饰所有字段
    - 不提供修改方法，构造函数一次性初始化
    - 典型：`String`、`Integer` 等包装类、枚举
- 优点：性能最高，无需任何同步开销
- 缺点：灵活性差，每次修改都要创建新对象

### **使用线程安全类**

`JDK`提供了大量线程安全的工具类，开箱即用，优先使用这些成熟实现，

| 分类 | 推荐使用 | 替代的非线程安全类 |
| :---- | :---- | :---- |
| 集合 | `ConcurrentHashMap`、`CopyOnWriteArrayList`、`CopyOnWriteArraySet` | `HashMap`、`ArrayList`、`HashSet` |
| 原子类 | `AtomicInteger`、`AtomicLong`、`AtomicReference` | 普通基本类型/引用 |
| 计数器 | `LongAdder`、`DoubleAdder` | `AtomicLong`（高并发下性能更好） |
| 同步工具 | `CountDownLatch`、`CyclicBarrier`、`Semaphore` | 手写等待/通知 |

### **volatile关键字**

轻量级同步
- 解决的问题：可见性和有序性（禁止指令重排序）
- 不能解决的问题：原子性（比如`count++`这种复合操作）
- 典型使用场景
    - 状态标记位（`boolean flag = true;`）
    - 双重检查锁（`DCL`）单例模式
    - 发布不可变对象

```java
// DCL单例模式的正确写法
public class Singleton {
    private static volatile Singleton instance; // 必须加volatile
    
    private Singleton() {}
    
    public static Singleton getInstance() {
        if (instance == null) { // 第一次检查
            synchronized (Singleton.class) { // 加锁
                if (instance == null) { // 第二次检查
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

>Q: `DCL`双重检查锁为什么要加`volatile`？<br/>
A: `JVM`会发生指令重排序：`1‑3‑2`。不加`volatile`，其他线程可能拿到半初始化对象。`volatile`禁止指令重排序，保证`1‑2‑3`执行顺序。`new Singleton()` 不是原子操作，分为三步，
1. 分配对象内存
2. 对象初始化构造函数
3. `instance`引用指向内存地址
{: .prompt-tip }

>Q: `volatile`可以修饰数组吗？比如`volatile int[] arr;`<br/>
A: 可以修饰数组引用，只保证数组引用的可见性，数组里面的元素不具备`volatile`语义，
- `arr = new int[10];` 引用赋值，`volatile`生效；
- `arr[0]=100;` 修改数组内部元素，`volatile`不生效；
- 想要数组元素原子/可见，使用`AtomicIntegerArray`。
{: .prompt-tip }

### **锁机制**

最常用的同步手段
1. `synchronized`关键字
- 特点：`JVM`内置锁，自动加锁和释放锁
- 锁的粒度：
    - 实例方法锁：锁当前对象
    - 静态方法锁：锁`Class`对象
    - 代码块锁：锁指定对象
- 优化：`JDK1.6`引入偏向锁、轻量级锁、重量级锁的升级机制
2. `java.util.concurrent.locks`包下的显式锁
- `ReentrantLock`：可重入锁，比`synchronized`更灵活
    - 支持公平锁/非公平锁
    - 可中断锁
    - 可尝试获取锁（`tryLock()`）
- `ReentrantReadWriteLock`：读写锁
    - 读‑读不互斥，读‑写互斥，写‑写互斥
    - 适合读多写少的场景，大幅提升并发性能

### **线程封闭**

隔离共享资源
- 原理：让数据只在一个线程内访问，从根本上避免竞争
- 实现方式
    1. 栈封闭：局部变量天然线程封闭
    2. `ThreadLocal`：每个线程拥有自己的变量副本

```java
private static ThreadLocal<SimpleDateFormat> dateFormatThreadLocal =
    ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));
```

>注意：`ThreadLocal`使用不当会导致内存泄漏，必须在`finally`中调用`remove()`。
{: .prompt-tip }

### **各种方法的性能与适用场景对比**

坐标轴，纵向表示代码复杂度从低到高逐渐升高；横向表示性能从左到右逐渐降低。

象限标注
- 左上：特定场景使用
- 右上：尽量避免
- 左下：推荐优先使用
- 右下：权衡使用

完整整理（二维视图含义）
- 左下（推荐优先使用，性能高、复杂度低）：无状态对象、不可变对象、线程安全类、`volatile`、原子类
- 左上（特定场景使用，性能尚可，复杂度偏高）：`ThreadLocal`、`ReentrantLock`、读写锁
- 右下（权衡使用，性能偏低）：`synchronized`
- 右上：尽量避免

```markdown
┌─────────────────────────────────────────────────────────────────────┐
│                          代码复杂度 高 ↑                             │
│                                                                     │
│  特定场景使用                       │                      尽量避免   │
│                                    │                                │
│                                    │                                │
│  读写锁                             │                                │
│                                    │                                │
│  ReentrantLock                     │                                │
│                                    │                                │
│  ThreadLocal                       │                                │
│                                    │                                │
│  原子类                             │                                │
│  volatile                          │                                │
│  线程安全类                         │                                │
│                                    │                                │
│  不可变对象                         │                                │
│  无状态对象                         │                                │
│────────────────────────────────────┼────────────────────────────────│
│                                    │                                │
│                                    │                                │
│                                    │                    synchronized│
│                                    │                                │
│                                    │                                │
│                                    │                                │
│                                    │                                │
│                                    │                                │
│                                    │                                │
│                                    │                                │
│                                    │                                │
│  推荐优先使用                       │                      权衡使用   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
                                  性能 低 →
```

| 优先级 | 方法名称 | 性能指数 | 代码复杂度 | 推荐指数 |
| :---- | :---- | :---- | :---- | :---- |
| 1 | 无状态对象 | ⭐⭐⭐⭐⭐ | ⭐ | ⭐⭐⭐⭐⭐ |
| 2 | 不可变对象 | ⭐⭐⭐⭐⭐ | ⭐ | ⭐⭐⭐⭐⭐ |
| 3 | `volatile`关键字 | ⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ |
| 4 | 原子类(`Atomic`系列) | ⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ |
| 5 | `JUC`线程安全类 | ⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ |
| 6 | `ThreadLocal` | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ |
| 7 | 读写锁 | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| 8 | `ReentrantLock` | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ |
| 9 | `synchronized` | ⭐⭐ | ⭐⭐⭐ | ⭐⭐ |

`Java`线程安全方案选型优先级：优先无状态/不可变对象，尽量避免直接使用重量级锁。

选型
1. 优先使用高层抽象：能用线程安全类就不用自己写锁
2. 最小化锁的范围：只在必要的代码块上加锁
3. 读写分离：读多写少场景优先使用读写锁
4. 避免死锁：按顺序获取锁、设置超时时间、使用tryLock
5. 并发工具优先：优先使用JUC包下的工具，不要重复造轮子

保证线程安全的优先级：无状态 > 不可变 > 线程安全类 > `volatile` > 显式锁 > `synchronized`。

实际开发中，要根据具体场景选择最合适的方案，在性能和复杂度之间找到最佳平衡点。

## **技术亮点代码实现**

### **不可变对象的正确实现**

```java
// 技术亮点：防止子类破坏不可变性、保证所有字段的可见性
public final class ImmutableUser { // 1. 类用final修饰，禁止继承
    private final Long id; // 2. 所有字段用final修饰
    private final String name;
    private final Date birthday; // 3. 对可变字段进行防御性拷贝

    public ImmutableUser(Long id, String name, Date birthday) {
        this.id = id;
        this.name = name;
        // 关键：不直接引用外部传入的可变对象
        this.birthday = new Date(birthday.getTime());
    }

    // 4. 不提供任何setter方法
    public Long getId() { return id; }
    public String getName() { return name; }

    // 5. getter返回可变对象的拷贝，而不是原引用
    public Date getBirthday() {
        return new Date(birthday.getTime());
    }
}
```

实现不可变对象的`5`条要点
1. 类使用`final`修饰，禁止被继承
2. 所有成员变量使用`final`修饰
3. 构造函数中，对传入的可变对象做防御性拷贝，不直接保存外部引用
4. 不提供`setter`修改方法
5. `getter`方法返回可变对象副本，不返回原始对象引用

### **CAS原子操作原理（AtomicInteger源码核心）**

```java
// 技术亮点：无锁编程、CPU原语支持、乐观锁思想
public class AtomicInteger extends Number implements java.io.Serializable {
    private volatile int value; // 保证可见性

    public final int getAndIncrement() {
        // CAS操作: compareAndSwapInt是native方法，由CPU指令保证原子性
        return unsafe.getAndAddInt(this, valueOffset, 1);
    }

    // Unsafe类中的核心方法
    // public final native boolean compareAndSwapInt(
    //     Object obj, long offset, int expect, int update);
}

// 手写一个简单的自旋锁
public class SpinLock {
    private AtomicBoolean locked = new AtomicBoolean(false);

    public void lock() {
        // 循环尝试获取锁，直到成功
        while (!locked.compareAndSet(false, true)) {
            // 空自旋，消耗CPU但避免线程上下文切换
        }
    }

    public void unlock() {
        locked.set(false);
    }
}
```

要点总结
1. `volatile`保证变量可见性；`CAS`（`Compare And Swap`）依靠`Unsafe`的`native`方法，`CPU`硬件指令保证原子性，属于乐观无锁。
2. `CAS`三要素：内存地址、预期值、新值；只有当前值等于预期值，才更新为新值。
3. 自旋锁：循环`CAS`抢锁，不阻塞线程；缺点是空自旋消耗`CPU`。
4. `CAS`经典问题：`ABA`问题、循环开销、只能保证单个变量原子性。

### **ReentrantLock正确使用范式（避免死锁）**

```java
// 技术亮点：可中断、可超时、公平锁支持
public class ReentrantLockExample {
    private final ReentrantLock lock = new ReentrantLock(true); // 公平锁

    public void doSomething() {
        lock.lock(); // 1. 加锁
        try {
            // 2. 业务逻辑（必须在try块内）
            System.out.println("执行业务逻辑");
        } finally {
            // 3. 必须在finally中释放锁！！！
            lock.unlock();
        }
    }

    // 更安全的写法: tryLock避免死锁
    public boolean tryDoSomething(long timeout, TimeUnit unit) throws InterruptedException {
        if (lock.tryLock(timeout, unit)) {
            try {
                System.out.println("获取锁成功，执行业务");
                return true;
            } finally {
                lock.unlock();
            }
        }
        // 获取锁失败
        return false;
    }
}
```

使用要点
1. `lock()`加锁写在`try`外面，业务逻辑放`try`内部；`unlock()`必须放在`finally`，保证锁一定释放。
2. 构造参数`true`=公平锁；`false`（默认）=非公平锁。
3. `tryLock(timeout,unit)`：带超时获取锁，防止无限等待死锁。
4. 支持锁中断`lockInterruptibly()`，`synchronized`不支持。

### **读写锁高性能实现（读多写少场景神器）**

```java
// 技术亮点：读写分离、读‑读不互斥、性能提升10倍以上
public class ReadWriteLockCache<K, V> {
    private final Map<K, V> cache = new HashMap<>();
    private final ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
    private final Lock readLock = rwLock.readLock();
    private final Lock writeLock = rwLock.writeLock();

    // 读操作：加读锁，多个线程可以同时读
    public V get(K key) {
        readLock.lock();
        try {
            return cache.get(key);
        } finally {
            readLock.unlock();
        }
    }

    // 写操作：加写锁，阻塞所有读和写
    public void put(K key, V value) {
        writeLock.lock();
        try {
            cache.put(key, value);
        } finally {
            writeLock.unlock();
        }
    }

    // 缓存更新：先写后读的原子操作
    public V putIfAbsent(K key, V value) {
        writeLock.lock();
        try {
            if (!cache.containsKey(key)) {
                cache.put(key, value);
                return null;
            }
            return cache.get(key);
        } finally {
            writeLock.unlock();
        }
    }
}
```

读写锁核心规则
1. 读‑读共享：多个线程可同时获取读锁，并发读；
2. 读‑写互斥：有读锁时写锁阻塞；有写锁时读锁阻塞；
3. 写‑写互斥：写锁独占；
4. 适用场景：读多写少，例如缓存；写频繁场景不适合；
5. 潜在问题：读锁过多可能造成写饥饿。

### **ThreadLocal正确使用（避免内存泄漏）**

```java
// 技术亮点：线程封闭、避免参数传递、SimpleDateFormat线程安全问题解决
public class ThreadLocalExample {

    // 正确写法：使用static final修饰ThreadLocal
    private static final ThreadLocal<SimpleDateFormat> DATE_FORMAT =
            ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));

    public String formatDate(Date date) {
        try {
            return DATE_FORMAT.get().format(date);
        } finally {
            // 关键：使用完必须remove()！！！防止内存泄漏
            DATE_FORMAT.remove();
        }
    }
}
```

`ThreadLocal`要点
1. 线程封闭：数据只属于当前线程，线程间互不共享，天然线程安全。
2. 定义：推荐`static final`，避免频繁创建`ThreadLocal`实例。
3. 内存泄漏根源
   - `ThreadLocalMap`的`key`是弱引用，`GC`会回收`key`；但`value`是强引用。
   - 线程池场景下线程复用，线程不销毁，`value`持续滞留，造成内存泄漏。
4. 最佳实践：`get()`使用完毕，`finally`块中必须调用`remove()`。
5. 典型场景：解决非线程安全工具类（`SimpleDateFormat`）、链路透传上下文。

>注意：不要用`set(null)`，只是把`value`设为`null`，不会清除`Entry`；必须调用`remove()`。
{: .prompt-tip }

## **技术难点与解决方案**

|技术难点|问题表现|根本原因|最佳解决方案|
|:---|:---|:---|:---|
|死锁问题|程序卡死，`CPU`使用率低，线程`BLOCKED`状态|多个线程循环等待对方持有的锁|1. 按固定顺序获取锁<br>2. 使用`tryLock()`设置超时<br>3. 避免一个线程同时持有多个锁<br>4. 使用`jstack`命令排查死锁|
|可见性问题|一个线程修改了变量，另一个线程看不到|`CPU`缓存与主内存不一致|1. 使用`volatile`关键字<br>2. 使用`synchronized`或`Lock`<br>3. 使用原子类|
|指令重排序问题|`DCL`单例模式返回`null`、对象半初始化|`JVM`为了优化性能对指令进行重排序|1. 使用`volatile`禁止指令重排序<br>2. 使用静态内部类实现单例|
|原子性问题|`count++`结果不正确、超卖问题|复合操作不是原子性的|1. 使用原子类(`AtomicInteger`)<br>2. 使用`synchronized`或`Lock`<br>3. 使用分段锁(`LongAdder`)|
|`ThreadLocal`内存泄漏|`OOM`异常、内存占用持续升高|`ThreadLocalMap`的`Entry`是弱引用，`Value`是强引用|1. 使用`static final`修饰`ThreadLocal`<br>2. 必须在`finally`中调用`remove()`<br>3. 避免存储大对象|
|锁性能问题|并发量上不去、吞吐量低|锁竞争激烈、锁粒度过大|1. 缩小锁的范围（只锁必要代码）<br>2. 使用读写锁（读多写少）<br>3. 使用分段锁(`ConcurrentHashMap`)<br>4. 使用无锁编程(`CAS`)|
|并发集合陷阱|`ConcurrentHashMap`的`putIfAbsent`误用|复合操作不保证原子性|1. 使用原子方法（`putIfAbsent`、`compute`）<br>2. 不要先检查再操作<br>3. 避免在遍历中修改集合|
|伪共享问题|多线程操作数组性能极差|多个变量位于同一个`CPU`缓存行|1. 使用缓存行填充（`@Contended`注解）<br>2. 避免多个线程频繁操作相邻变量|

关键概念小结
1. 死锁四条件：互斥、占有且等待、不可剥夺、循环等待；破坏任意一条即可避免死锁；
2. 并发三大问题：原子性、可见性、有序性；
3. 伪共享：多个线程修改同一个`CPU`缓存行内不同变量，触发频繁缓存失效，性能暴跌。`@Contended`用来填充缓存行规避；
4. `ConcurrentHashMap`只是单个方法线程安全，`if‑contains‑then‑put`这种复合操作仍然需要调用内置原子API，不能自己先判断再写入。

### **静态内部类单例模式（完美解决DCL问题）**

```java
// 技术亮点：懒加载、线程安全、无锁、性能最高
public class Singleton {
    private Singleton() {} // 私有构造函数

    // 静态内部类，只有在第一次被调用时才会加载
    private static class SingletonHolder {
        private static final Singleton INSTANCE = new Singleton();
    }

    public static Singleton getInstance() {
        return SingletonHolder.INSTANCE;
    }
}
```

>原理：利用`JVM`类加载机制保证初始化线程安全；外部类加载不会加载内部类，实现懒加载，不需要`volatile`，避免`DCL`指令重排序问题。
{: .prompt-tip }

### **LongAdder分段锁思想（高并发计数器首选）**

```java
// 技术亮点：分段锁、减少CAS竞争、高并发下性能比AtomicLong高10倍
public class LongAdder extends Striped64 implements Serializable {
    // 核心思想: 将一个Long值拆分成多个Cell
    // 不同线程更新不同的Cell，最后求和
    // 只有当Cell竞争失败时才会扩容

    public void add(long x) {
        Cell[] as; long b, v; int m; Cell a;
        if ((as = cells) != null || !casBase(b = base, b + x)) {
            boolean uncontended = true;
            if (as == null || (m = as.length - 1) < 0 ||
                    (a = as[getProbe() & m]) == null ||
                    !(uncontended = a.cas(v = a.value, v + x)))
                longAccumulate(x, null, uncontended);
        }
    }

    public long sum() {
        Cell[] as = cells;
        long sum = base;
        if (as != null) {
            for (Cell a : as)
                if (a != null)
                    sum += a.value;
        }
        return sum;
    }
}
```

要点
1. 把计数分散到多个`Cell`数组，线程分散在不同`Cell`做`CAS`，降低竞争；
2. `sum()`只是把`base+`所有`Cell`累加，不保证实时一致性，适合统计计数，不适合强一致性场景。

### **小结**

线程安全的核心是正确处理原子性、可见性、有序性三个问题。在实际开发中，我们应该遵循**能不用锁就不用锁，能用高层抽象就不用底层实现**的原则，优先选择无状态、不可变对象和`JUC`包提供的成熟工具类，只有在必要时才使用锁机制。

>Q: 保证线程安全的方法有哪些？
{: .prompt-tip }

A: 线程不安全本质上是，多个线程同时写同一个可变资源导致的竞态条件。

解决方案一般分成三大类，
1. 互斥同步（悲观锁）：同一时间只让一个线程操作，比如`synchronized`、`ReentrantLock`。
2. 非阻塞同步（乐观锁）：基于`CAS`的原子类，如`AtomicInteger`。
3. 无同步方案：干脆不共享，比如不可变对象、`ThreadLocal`、栈封闭。

这三类就像处理多人抢卫生间：第一类是排队加锁，第二类是大家先上，冲突再协调，第三类是给每人发个独立卫生间。

>Q: `synchronized`和`ReentrantLock`有什么本质区别？什么时候你选后者？
{: .prompt-tip }

A: 区别如下，
- `synchronized`是`JVM`内置锁，自动释放，代码简单，但是功能上比较憨，没法中途放弃；
- `ReentrantLock`是`API`层面的锁，灵活，可以`tryLock`带超时、可中断、可设公平锁，还能绑定多个`Condition`做精准唤醒。

选后者的场景，比如调用一个外部接口，我不希望线程傻等，就用`tryLock(200, TimeUnit.MILLISECONDS)`，拿不到锁直接降级。这种场合`synchronized`就做不到。

>Q: 抓住了灵活性的关键点。读写锁你了解吗？什么情况下它比独占锁性能好？
{: .prompt-tip }

A: 读写锁就是`ReentrantReadWriteLock`，它把锁分成读锁（共享锁）和写锁（独占锁）。读和读不互斥，读和写互斥。

典型场景：配置中心，读配置的线程非常多，写配置很少。如果用`synchronized`，所有读都互斥，吞吐量上不去。换成读写锁后，大部分读操作可以并发，只有写时才阻塞所有人。当然，用时要小心锁降级的规则。

>Q: 非阻塞同步，`CAS`是什么原理？原子类为什么能无锁却能线程安全？
{: .prompt-tip }

A: `CAS`就是`Compare‑And‑Swap`，是`CPU`级别的原子指令。好比你去改一个值，改之前先核对一下手里的旧值是不是跟主存里一致，一致就换上新值，不一致就说明别人改过，重新读再试。

`Java`里的`AtomicInteger.incrementAndGet()`内部就是循环`CAS`，直到成功，不依赖操作系统挂起线程，所以高并发下吞吐量更高，没有上下文切换开销。不过它有`ABA`问题，可以用`AtomicStampedReference`加版本号解决。

>Q: 无同步方案，怎么才能做到不共享？
{: .prompt-tip }

A: 核心思想是彻底消灭共享资源，有三个典型手段，
1. 不可变对象，比如`String`，一经创建状态不可改。自己写类用`final`修饰`class`，所有字段`private final`，不提供`setter`。这种对象天生线程安全，随便多线程读。
2. `ThreadLocal`给每个线程一个独立的数据副本，线程之间物理隔离。像`Spring`的`RequestContextHolder`就用`ThreadLocal`保存当前请求。用完后必须`remove()`，否则在线程池场景下会内存泄漏。
3. 栈封闭方法里的局部变量存在每个线程自己栈里，绝无共享，自然安全。只要别把局部变量的引用发布出去就行。

另外，无状态设计也算：一个`Service`没有成员变量，所有逻辑都在方法参数和局部变量上，多线程调用天然安全。

>Q: 快速画个全景图，把今天聊的这些串起来。
{: .prompt-tip }

A: 
```
保证线程安全
├─ 互斥同步（悲观锁）
│  ├─ synchronized（内置锁，自动）
│  ├─ ReentrantLock（显式锁，灵活）
│  └─ ReadWriteLock/StampedLock（读多写少）
├─ 非阻塞同步（乐观锁）
│  └─ CAS 原子类（AtomicInteger, AtomicReference）
└─ 无同步方案（避免共享）
   ├─ 不可变对象（String, Integer）
   ├─ ThreadLocal（线程私有副本）
   ├─ 栈封闭（局部变量）
   └─ 无状态设计（无成员变量）
```

>Q: 看个场景，如果给一个全局的统计接口调用次数设计线程安全的计数器，读写比大概`100:1`，你选哪个方法？
{: .prompt-tip }

A: 读写比极高，明显读远大于写。

方案一：直接用`AtomicLong`，读非常快（无锁），写也是`CAS`轻量自旋，足够。

但如果读的是很复杂的衍生数据，我可能会用`ReentrantReadWriteLock`，让大量读线程完全并发。不过对于简单计数器，`AtomicLong`更简洁，实测性能也极好。再补一句，`JDK8`还提供了`LongAdder`，高并发写时有更好的分散热点设计。

>Q: 刚才提到的几种同步方式，写几段核心代码，把最有技术亮点的用法展示出来？顺便说说为什么这么写有亮点。
{: .prompt-tip }

A: 这里写三段代码，分别对应互斥同步的高级用法、非阻塞同步的精髓以及无同步方案里最容易踩坑的点。

1. `ReentrantLock`的`tryLock`超时与降级处理

    ```java
    public class TimeoutRetryService {
        private final Lock lock = new ReentrantLock();

        public boolean processWithTimeout(String orderId) {
            boolean acquired = false;
            try {
                // 技术亮点：尝试获取锁，等200ms拿不到就降级，不阻塞
                acquired = lock.tryLock(200, TimeUnit.MILLISECONDS);
                if (!acquired) {
                    // 降级策略：放入延迟队列，或返回操作进行中
                    System.out.println("系统繁忙，订单 " + orderId + " 稍后重试");
                    return false;
                }
                // 临界区：真正处理业务
                doActualWork(orderId);
                return true;
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                return false;
            } finally {
                // 亮点：防止意外释放未持有的锁
                if (acquired) {
                    lock.unlock();
                }
            }
        }

        private void doActualWork(String orderId) {
            // 模拟幂等操作
        }
    }
    ```

    亮点说明
    - `tryLock`带超时 + 降级，避免线程饿死在锁上。
    - `finally`中通过`acquired`标志判断是否成功获取，防止`IllegalMonitorStateException`，这是工程里很常见的健壮性写法。

2. `AtomicInteger`的`CAS`循环（高性能计数器）

    ```java
    public class CasCounter {
        private final AtomicInteger count = new AtomicInteger(0);

        public int incrementAndGet() {
            int prev, next;
            do {
                prev = count.get();
                next = prev + 1;
                // 亮点: CAS自旋，无锁，高并发下吞吐量远超synchronized
            } while (!count.compareAndSet(prev, next));
            return next;
        }

        // JDK8 提供的更简洁写法，内部就是 CAS
        public int quickIncrement() {
            return count.incrementAndGet(); // 一行搞定
        }
    }
    ```

    亮点说明
    - 手动展示了`CAS`循环的原始逻辑，不要只会用`API`，得懂原理。
    - 对比 `incrementAndGet()`，说明底层也是`CAS`，上层才是原子包装。

3. `ThreadLocal`的正确使用与内存泄漏预防

    ```java
    public class RequestContextHolder {
        private static final ThreadLocal<Map<String, Object>> CONTEXT =
                ThreadLocal.withInitial(HashMap::new);

        public static void set(String key, Object value) {
            CONTEXT.get().put(key, value);
        }

        public static Object get(String key) {
            return CONTEXT.get().get(key);
        }

        // ✨核心亮点: 显式提供清理方法，防止线程池复用导致内存泄漏
        public static void clear() {
            CONTEXT.remove(); // 必须调!
        }
    }

    // 使用方: 在过滤器 finally 块中清理
    try {
        RequestContextHolder.set("userId", "10086");
        // ... 业务处理
    } finally {
        RequestContextHolder.clear(); // ⚠️ 关键一步
    }
    ```

    亮点说明
    - `ThreadLocal.withInitial()` 提供初始值，代码更优雅；
    - 重点强调了`remove()`，这是生产环境中内存泄漏的高发点，能提到它说明你有`JVM`调优意识。

>Q: 总结一下，在保证线程安全的设计中，遇到过哪些技术难点，分别怎么解决？
{: .prompt-tip }

A: 总结五个最常见的难点和方案，

|技术难点|典型场景|解决方案|
| :---- | :---- | :---- |
|复合操作的原子性|`if(!map.containsKey(key)) map.put(...)` 这样的检查‑执行操作|用 `ConcurrentHashMap.putIfAbsent()`，或把操作包在`synchronized`块中，但优先用并发容器的原子方法|
|锁粒度控制与死锁|大锁导致性能差，多锁嵌套导致死锁|缩小同步范围（快进快出），统一加锁顺序，用`tryLock`超时预防死锁，或用`jstack`检测|
|`CAS`的`ABA`问题|链表头节点从`A-B-A`，虽然值相同但状态已变|使用带版本号的`AtomicStampedReference`或`AtomicMarkableReference`，在链表操作中尤其关键|
|`ThreadLocal`内存泄漏|线程池线程复用，`ThreadLocal`引用不清理导致`OOM`|强制在`finally`中调用`remove()`，或者使用带弱引用的自定义`ThreadLocal`，建议所有框架都内置清理|
|可见性&指令重排|一个线程改了值，另一个线程永远看不见|用`volatile`保证可见性和禁止指令重排，或者使用锁（隐式内存屏障），避免依赖运气|

>Q: `synchronized`锁升级过程？
{: .prompt-tip }

`JDK1.6`之后锁会自动升级：偏向锁 → 轻量级锁 → 重量级锁，锁只能升级，不能降级。
1. 偏向锁：只有一个线程竞争，消除`CAS`；线程获取锁，记录线程`ID`。
2. 轻量级锁：出现多线程竞争，撤销偏向锁；自旋`CAS`抢锁，不阻塞。
3. 重量级锁：自旋失败膨胀为重量级锁；依赖`OS`互斥锁，线程阻塞。

偏向锁可以通过`-XX:-UseBiasedLocking`参数关闭。

>Q: `AtomicInteger`和`LongAdder`的区别？
{: .prompt-tip }

A: 区别如下，
1. `AtomicInteger`：无限循环`CAS`更新单个变量；竞争大时大量自旋，`CPU`消耗高；
2. `LongAdder`：分段`Cell`数组，竞争分散到不同`Cell`；最后求和。高并发写性能更强。

注意：`LongAdder sum()`是合并计算，不保证强一致性；适合统计计数，不适合需要实时准确读取的业务。

>Q: `ConcurrentHashMap 1.7`与`1.8`的区别
{: .prompt-tip }

A:
- `1.7`：分段锁`Segment`，`Segment`继承`ReentrantLock`；分段加锁，降低锁粒度。
- `1.8`：废弃`Segment`，`CAS + synchronized`；锁粒度降到`Node`节点；
    - 读操作无锁；
    - 写：`CAS`成功直接写入；`CAS`失败，对`Node`加`synchronized`锁。
    - 红黑树：链表长度大于`8`转为红黑树，小于`6`退化为链表。

为什么放弃`Segment`：`Segment`内存占用大；`synchronized`经过优化，性能不差。

>Q: 什么是锁降级？读写锁是否支持锁升级？
{: .prompt-tip }

A: 
- 锁降级：写锁 → 读锁，允许；
- 锁升级：读锁 → 写锁，禁止，会发生死锁。

1. 锁降级：指线程在持有写锁的前提下，先获取读锁，随后释放写锁，完成从写锁到读锁的转换，读写锁是支持锁降级的。
- 核心流程：持有写锁 → 获取读锁 → 释放写锁，最终保留读锁，完成锁降级。
2. 锁升级：指线程持有读锁，尝试进一步获取写锁，读写锁不支持锁升级。
- 原因：线程已经持有读锁，再申请写锁时，写锁需要等待所有读锁释放；但当前线程本身持有的读锁没有释放，写锁一直等待，造成自死锁，线程会永久阻塞。

>Q: `AQS`简单说下原理？
{: .prompt-tip }

A: `AQS`，`AbstractQueuedSynchronizer`，`JUC`锁的底层基础。
- 核心：`state`状态变量（`volatile`）、双向`FIFO`等待队列。
- `state`：`0`未加锁；大于`0`代表已占有锁。
- 线程抢锁失败进入`CLH`队列阻塞；锁释放唤醒队列后继线程。
- 分为独占模式（`ReentrantLock`）、共享模式（读写锁读锁、`CountDownLatch`）。

>Q: `sleep()`和`wait()`的区别
{: .prompt-tip }

A:
1. `sleep()`是`Thread`静态方法；`wait()`是`Object方`法。
2. `sleep()`不释放锁；`wait()`必须释放锁。
3. `sleep()`时间到自动恢复；`wait()`需要`notify/notifyAll`唤醒。
4. `sleep`不需要`synchronized`环境；`wait`必须在`synchronized`同步块内调用。

>Q: 为什么线程池不建议使用`Executors`创建？
{: .prompt-tip }

A: 原因如下，
- `newFixedThreadPool`/`newSingleThreadExecutor`：队列`LinkedBlockingQueue`无界，任务堆积`OOM`；
- `newCachedThreadPool`：最大线程数`Integer.MAX_VALUE`，创建大量线程耗尽`CPU`。

推荐手动`ThreadPoolExecutor`构造函数创建，自定义队列、拒绝策略。

> Q: `Java`线程池参数、拒绝策略
{: .prompt-tip }

A: 七大参数，
1. `corePoolSize`：核心线程数
2. `maximumPoolSize`：最大线程数
3. `keepAliveTime`：空闲线程存活时间
4. `unit`：时间单位
5. `workQueue`：阻塞队列
6. `threadFactory`：线程工厂
7. `handler`：拒绝策略

四种拒绝策略，
1. `AbortPolicy`：抛出异常（默认）
2. `DiscardPolicy`：直接丢弃任务，不抛异常
3. `DiscardOldestPolicy`：丢弃队列最旧任务，尝试执行当前任务
4. `CallerRunsPolicy`：调用者线程执行任务

## **附录**

### **Java对象结构**

不同`JVM`的对象结构，实现不一样，这里以`HotSpot JVM`为例。

#### **oop‑klass模型**

`HotSpot JVM`并没有将`Java`实例对象直接一对一的映射到本地（`native`）的`C++`对象，而是设计了一个`oop‑klass`模型。

什么是`OOP`呢？`OOP`（`Ordinary Object Pointer`，普通对象指针）是指对象—类二者中的对象，表示对象的实例信息，从名字看是一个指针，实际并不仅仅是一个内存地址，而是对内存地址的一个描述或者对内存中数据结构的一个描述。

`JVM`中的对象的类被定义为`oopDesc`，为了区别于`Java`语言中的`Object`对象，`JVM`对象实例的`C++`类型为`instanceOopDesc`，其基类为`oopDesc`，代码如下，

```cpp
class oopDesc {
  friend class VMStructs;
 private:
  volatile markOop  _mark;                  //对象头
  union _metadata {
    wideKlassOop    _klass;                 //普通指针
    narrowOop       _compressed_klass;      //压缩类指针
  } _metadata;
 private:
  //省略不相干的代码
}

class instanceOopDesc : public oopDesc {   //普通对象类型
  //省略不相干的代码
}

class arrayOopDesc : public oopDesc {      //数组对象类型
  //省略不相干的代码
}
```

每当在`Java`代码中创建一个对象时，`JVM`会创建一个`instanceOopDesc`实例来表示这个对象，此对象实例存放在堆区。类似地，每当在`Java`代码中创建一个数组时，`JVM`会创建一个`arrayOopDesc`实例来表示。所以，一个普通`Java`对象的底层为一个`instanceOopDesc`实例。

在`oop‑klass`模型中什么是`Klass`？`Klass`指的是对象—类二者中的类。为了区别于`Java`语言的`Class`类，`JVM`中用`Klass`来描述类型，`Klass`包含元数据和方法信息，用来描述语言层的类型。

```cpp
//用来描述语言层的类型
class Klass : public Metadata {
    //省略不相干的代码
    //指向java.lang.Class的instance，mirroring this class即是这个类的影子类
    OopHandle _java_mirror;
}

//在虚拟机层面描述一个Java类
class InstanceKlass: public Klass {
    //省略不相干的代码
}
```

`HotSpot`为每一个已加载的`Java`类创建一个`InstanceKlass`对象，用来在`JVM`层表示`Java`元数据对象。但是这个`InstanceKlass`对象就是给`JVM`内部用的，并不直接暴露给`Java`层。实际上，给`Java`层用的类元数据对象为`java.lang.Class`类型的对象，或者说`java.lang.Class`类型的实例。

那么，一份类的元数据就出现了两个对象，一个是`Java`层的`java.lang.Class`类型的实例；一个`JVM`层的`InstanceKlass`类型的实例。

根据前面的`Java`对象的底层介绍，一个普通`Java`对象的底层为一个`instanceOopDesc`实例。我们知道，`Java`层的`java.lang.Class`类型的实例也是一个普通对象，所以`Class`对象也就对应到一个`instanceOopDesc`实例。这个`instanceOopDesc`实例，被称为`JVM`层`InstanceKlass`实例的`Java`镜像，二者的关系如下图，

![Desktop View](/assets/images/20260829/jvm_klass_mirror.png){: width="600" height="300" }
_JVM堆区和方法区_

`InstanceKlass`实例可以导航到其`Java`镜像，具体的成员为`_java_mirror`，可以导航到`instanceOopDesc`实例，也就是`java.lang.Class`类型的实例。

大致了解`oop‑klass`模型后，接下来就好介绍`Java`对象（`Object`实例）结构了，其实际上是`C++`中`instanceOopDesc`的结构。

#### **Java对象结构**

`Java`对象（`Object`实例）结构包括三部分，分别是对象头、对象体和对齐字节，具体如下图，

![Desktop View](/assets/images/20260829/java_obj_struct.png){: width="600" height="300" }
_Java对象（Object实例）结构_

`Java`对象（`Object`实例）的三个部分
1. 对象头，包括三个字段，
- `Mark Word`（标记字），用于存储自身运行时的数据，例如`GC`标志位、哈希码、锁状态等信息。
- `Class Pointer`（类对象指针），用于存放此对象的元数据（`InstanceKlass`）的地址。虚拟机通过此指针可以确定这个对象是哪个类的实例。
- `Array Length`（数组长度），如果对象是一个`Java`数组，那么此字段必须有，用于记录数组长度的数据；如果对象不是一个`Java`数组，那么此字段不存在，所以这是一个可选字段。
2. 对象体
- 对象体包含了对象的实例变量（成员变量），用于成员属性值，包括父类的成员属性值。这部分内存按`4`字节对齐。
3. 对齐字节
- 对齐字节也叫作填充对齐，用来保证`Java`对象在所占内存字节数为`8`的倍数（`8N bytes`）。`HotSpot VM`的内存管理要求对象起始地址必须是`8`字节的整数倍。对象头本身是`8`的倍数，当对象的实例变量数据不是`8`的倍数，需要填充数据来保证`8`字节的对齐。

`Object`实例结构中几个重要的字段，做一下简要说明，
- `Mark Word`（标记字）字段主要用来表示对象的线程锁状态，另外还可以用来配合`GC`、存放该对象的`hashCode`。
- `Class Pointer`（类对象指针）字段是一个指向方法区中类元数据信息的指针，意味着该对象可随时知道自己是哪个`Class`的实例。
- `Array Length`（数组长度）字段也占用`32`位（在`32`位`JVM`中）的字节，这是可选的，只有当本对象是一个数组对象时才会有这个部分。
- 对象体用于保存对象属性值，是对象的主体部分，占用的内存空间大小取决于对象的属性数量和类型。
- 对齐字节并不是必然存在的，也没有特别的含义，它仅仅起着占位符的作用。当对象实例数据部分没有对齐（`8`字节的整数倍）时，就需要通过对齐填充来补全。

##### **对象结构中的字段长度**

`Mark Word`、`Class Pointer`、`Array Length`等字段的长度都与`JVM`的位数有关。`Mark Word`的长度为`JVM`的一个`Word`（字）大小，也就是说`32`位`JVM`的`Mark Word`为`32`位，`64`位`JVM`为`64`位。`Class Pointer`（类对象指针）字段的长度也为`JVM`的一个`Word`大小，即`32`位的`JVM`为`32`位，`64`位的`JVM`为`64`位。

对于对象指针而言，如果`JVM`中对象数量过多，使用`64`位的指针将浪费大量内存，通过简单统计，`64`位的`JVM`将会比`32`位的`JVM`多耗费`50%`的内存。为了节约内存可以使用选项`+UseCompressedOops`开启指针压缩。选项`UseCompressedOops`中的`Oop`部分为`Ordinary object pointer`（普通对象指针）的缩写。

如果开启`UseCompressedOops`选项，以下类型的指针将从`64`位压缩至`32`位，
- `Class`对象的属性指针（即静态变量）
- `Object`对象的属性指针（即成员变量）
- 普通对象数组的元素指针

当然，也不是所有的指针都会压缩，一些特殊类型的指针不会压缩，比如指向`PermGen`（永久代）的`Class`对象指针（`JDK 8`中指向元空间的`Class`对象指针）、本地变量、堆栈元素、入参、返回值和`NULL`指针等。

`Mark Word`的位长度也不会受到`OOP`对象指针压缩选项的影响。

#### **Mark Word的结构信息**

`Java`内置锁涉及的很多重要信息，这些都存放在对象结构中，并且存放于对象头的`Mark Word`字段中。

`Java`内置锁的状态总共有`4`种，级别由低到高依次为无锁、偏向锁、轻量级锁和重量级锁。在`JDK 1.6`之前，`Java`内置锁只是一个重量级锁，是一个效率比较低下的锁，在`JDK 1.6`之后，`JVM`为了提高锁的获取与释放效率，对`synchronized`的实现进行了优化，引入了偏向锁、轻量级锁的实现，从此以后`Java`内置锁的状态就有了`4`种（无锁、偏向锁、轻量级锁和重量级锁），并且`4`种状态会随着竞争的情况逐渐升级，而且是不可逆的过程，即不可降级，也就是说只能进行锁升级（从低级别到高级别）。

##### 不同锁状态下的Mark Word字段结构

`Mark Word`字段的结构与`Java`内置锁的状态强相关。为了让`Mark Word`字段存储更多的信息，`JVM`将`Mark Word`的最低两个位设置为`Java`内置锁状态位，不同锁状态下的`32`位`Mark Word`结构，如下表所示，

![Desktop View](/assets/images/20260829/mark_word_struct_32.png){: width="800" height="400" }
_不同锁状态下的32位Mark Word结构_

`64`位的`Mark Word`与`32`位的`Mark Word`结构相似，如下表所示，

![Desktop View](/assets/images/20260829/mark_word_struct_64.png){: width="800" height="400" }
_不同锁状态下的64位Mark Word结构_

目前主流的`JVM`都是`64`位，接下来对`64`位`Mark Word`中各部分的内容做具体介绍，
- `lock`：锁状态标记位，占两个二进制位，由于希望用尽可能少的二进制位表示尽可能多的信息，所以设置了`lock`标记。该标记的值不同，整个`Mark Word`表示的含义不同；
- `biased_lock`：对象是否启用偏向锁标记，只占`1`个二进制位。为`1`时表示对象启用偏向锁，为`0`时表示对象没有偏向锁；
- `age`：`4`位的`Java`对象分代年龄。在`GC`中，如果对象在`Survivor`区复制一次，年龄增加`1`。当对象达到设定的阈值时，将会晋升到老年代。默认情况下，并行`GC`的年龄阈值为`15`，并发`GC`的年龄阈值为`6`。由于`age`只有`4`位，因此最大值为`15`，这就是`‑XX:MaxTenuringThreshold`选项最大值为`15`的原因；
- `identity_hashcode`：`31`位的对象标识`HashCode`（哈希码）采用延迟加载技术，当调用`Object.hashCode()`方法或者`System.identityHashCode()`方法计算对象的`HashCode`后，其结果将被写到该对象头中。当对象被锁定时，该值会移动到`Monitor`（监视器）中；
- `thread`：`54`位的线程`ID`值为持有偏向锁的线程`ID`；
- `epoch`：偏向时间戳；
- `ptr_to_lock_record`：占`62`位，在轻量级锁的状态下指向栈帧中锁记录的指针；
- `ptr_to_heavyweight_monitor`：占`62`位，在重量级锁的状态下，指向对象监视器的指针。

`lock`和`biased_lock`两个标记位组合在一起，共同表示`Object`实例处于什么样的锁状态。二者组合的含义具体如下表所示，

|状 态|`biased_lock`|`lock`|
|:---|:---|:---|
|无锁|`0`|`01`|
|偏向锁|`1`|`01`|
|轻量级锁|`0`|`00`|
|重量级锁|`0`|`10`|
|GC标记|`0`|`11`|

### **Java内置锁**

>在`Java`世界里一切皆对象。`Java`有两种对象，分别是`Object`实例对象和`Class`对象。每个类运行时的类型信息用`Class`对象表示，它包含与类名称、继承关系、字段、方法有关的信息。`JVM`将一个类加载入自己的方法区内存时，会为其创建一个`Class`对象，对于一个类来说其`Class`对象也是唯一的。<br/>
`Class`类没有公共的构造方法，`Class`对象是在类加载时由`Java`虚拟机调用类加载器中的`defineClass`方法自动构造的，因此不能显式地声明一个`Class`对象。<br/>
所有的类都是在第一次使用时被动态加载到`JVM`中（懒加载），其各个类都是在必需时才加载的。这一点与许多传统语言（如`C++`）都不同，`JVM`为动态加载机制配套了一个判定动态加载使能的行为，使得类加载器首先检查这个类的`Class`对象是否已经被加载。如果尚未加载，类加载器会根据类的全限定名查找`.class`文件，验证后加载到`JVM`的方法区内存，并构造其对应的`Class`对象。
{: .prompt-tip }

每个`Java`对象都隐含有一把锁，就是`Java`内置锁（或者对象锁、隐式锁）。使用`synchronized(syncObject)`调用相当于获取`syncObject`的内置锁，所以可以使用内置锁对临界区代码段进行排他性保护。

`synchronized`同步方法，具体的例子如下，

```java
public class SafePlus
{
    private Integer amount = 0;
    //临界区代码段，使用synchronized进行保护
    public synchronized void selfPlus()
    {
        amount++;
    }
}
```

`synchronized`方法是一种粗粒度的并发控制，某一时刻只能有一个线程执行该`synchronized`方法；而`synchronized`代码块是一种细粒度的并发控制，处于`synchronized`块之外的其他代码是可以被多条线程并发访问的。在一个方法中，并不一定所有代码都是临界区代码段，可能只有几行代码会涉及线程同步问题。所以`synchronized`代码块比`synchronized`方法更细粒度地控制多线程的同步访问。

在`Java`内部实现上，`synchronized`同步方法等同于一个`synchronized`代码块，这个代码块包含了同步方法中的所有语句，然后在`synchronized`代码块的括号中传入`this`关键字，使用`this`对象锁作为进入临界区的同步锁。`synchronized`同步方法免去了手工设置同步锁的工作，这两种实现多线程同步的`plus`方法版本，编译成`JVM`内部字节码后结果是一样的，

```java
public void plus() {
    synchronized(this) { //对方法内部的全部代码进行保护
        amount++;
    }
}

public synchronized void plus() {
    amount++;
}
```

对于小的临界区，可以直接在方法声明中设置`synchronized`同步关键字，可以避免竞态条件（`Race Conditions`）的问题。但是对于较大的临界区代码段，为了执行效率，最好将同步方法分为小的临界区代码段。

```java
public class TwoPlus {

    private int sum1 = 0;
    private int sum2 = 0;
    //同步方法
    public synchronized void plus(int val1, int val2){
        //临界区代码段
        this.sum1 += val1;
        this.sum2 += val2;
    }
}
```

这里的临界区代码段包含了对两个临界区资源的操作，这两个临界区资源分别为`sum1`、`sum2`。使用`synchronized`对`plus(int val1, int val2)`进行同步保护之后，进入临界区代码段的线程拥有`sum1`、`sum2`的操作权，并且是全部占用。一旦线程进入，当线程在操作`sum1`而没有操作`sum2`时，也将`sum2`的操作权白白占用，其他的线程由于没有进入临界区，只能看着`sum2`被闲置而不能去执行操作。

`synchronized`同步块，具体的例子如下，

```java
public class TwoPlus{

    private int sum1 = 0;
    private int sum2 = 0;
    private Integer sum1Lock = new Integer(1);     // 同步锁一
    private Integer sum2Lock = new Integer(2);     // 同步锁二

    public void plus(int val1, int val2){
        //同步块1
        synchronized(this.sum1Lock){
            this.sum1 += val1;
        }
        //同步块2
        synchronized(this.sum2Lock){
            this.sum2 += val2;
        }
    }
}
```

在`TwoPlus`代码中，由于两个同步块保护着两个独立的临界区代码段，需要两把不同的`syncObject`对象锁，因此`TwoPlus`代码新加了`sum1Lock`和`sum2Lock`两个新的成员属性。这两个属性没有参与业务处理，`TwoPlus`仅仅利用了内置锁功能。

>如果某个`synchronized`方法是`static`（静态）方法，而不是普通的对象实例方法，其同步锁又是什么呢？<br/>
静态方法属于`Class`实例而不是单个`Object`实例，在静态方法内部不可以访问`Object`实例的`this`引用（也叫指针、句柄）。所以，修饰`static`静态方法`synchronized`关键字就没有办法获得`Object`实例的`this`对象的监视锁。
{: .prompt-tip }

使用`synchronized`关键字修饰`static`静态方法时，`synchronized`的同步锁并不是普通`Object`对象的监视锁，而是类所对应的`Class`对象的监视锁。下面将`Object`对象的监视锁叫作对象锁，将`Class`对象的监视锁叫作类锁。

`synchronized`静态同步方法如下，

```java
public class SafeStaticMethodPlus
{
    //静态的临界区资源
    private static Integer amount = 0;
    //使用synchronized关键字修饰static 静态方法
    public static synchronized void selfPlus()
    {
        amount++;
    }
}
```

由于类的对象实例可以有很多，但是每个类只有一个`Class`实例，所以使用类锁作为`synchronized`的同步锁时会造成同一个`JVM`内的所有线程只能互斥进入临界区段，是非常粗粒度的同步机制。

通过`synchronized`关键字所抢占的同步锁，什么时候释放呢？一种场景是`synchronized`块（代码块或者方法）正确执行完毕，监视锁自动释放；另一种场景是程序出现异常，非正常退出`synchronized`块，监视锁也会自动释放。所以，使用`synchronized`块时不必担心监视锁的释放问题。

`Java`内置锁的很多重要信息都存放在对象结构中，具体参见[附录-Java对象结构](https://shouyuanman.github.io/posts/java-thread-safe/#java%E5%AF%B9%E8%B1%A1%E7%BB%93%E6%9E%84)。

在`JDK 1.6`版本之前，所有的`Java`内置锁都是重量级锁。重量级锁会造成`CPU`在用户态和核心态之间频繁切换，代价高、效率低。`JDK 1.6`版本为了减少获得锁和释放锁所带来的性能消耗，引入了偏向锁和轻量级锁实现。

在`JDK 1.6`版本里，内置锁一共有`4`种状态，分别是无锁、偏向锁、轻量级锁和重量级锁，这些状态随着竞争情况逐渐升级。内置锁可以升级但不能降级，意味着偏向锁升级成轻量级锁后不能降级成偏向锁。这种能升级却不能降级的策略，其目的是为了提高获得锁和释放锁的效率。

#### **无锁状态**

`Java`对象刚创建时还没有任何线程来竞争，说明该对象处于无锁状态（无线程竞争它）这偏向锁标识位是`0`、锁状态`01`。无锁状态下对象的`Mark Word`如下所示，

![Desktop View](/assets/images/20260829/mark_word_no_lock.png){: width="600" height="300" }
_无锁状态下对象的Mark Word_

#### **偏向锁状态**

偏向锁是指一段同步代码一直被同一个线程所访问，那么该线程会自动获取锁，降低获取锁的代价。如果内置锁处于偏向状态，当有一个线程来竞争锁时，先用偏向锁，表示内置锁偏爱这个线程，这个线程要执行该锁关联的同步代码时，不需要再做任何检查和切换。偏向锁在竞争不激烈的情况下效率非常高。

偏向锁状态的`Mark Word`会记录内置锁自己偏爱的线程`ID`，内置锁会将该线程当作自己的熟人。偏向锁状态下对象的`Mark Word`具体如下图所示，

![Desktop View](/assets/images/20260829/mark_word_biased_lock.png){: width="600" height="300" }
_偏向锁状态下对象的Mark Word_

#### **轻量级锁状态**

当有两个线程开始竞争这个锁对象时，情况发生变化了，不再是偏向（独占）锁了，锁会升级为轻量级锁，两个线程公平竞争，哪个线程先占有锁对象，锁对象的`Mark Word`就指向哪个线程的栈帧中的锁记录。轻量级锁状态下对象的`Mark Word`如下图所示，

当锁处于偏向锁又被另一个线程所企图抢占时，偏向锁就会升级为轻量级锁。企图抢占的线程会通过自旋的形式尝试获取锁，不会阻塞抢锁线程，以便提高性能。

![Desktop View](/assets/images/20260829/mark_word_light_lock.png){: width="600" height="300" }
_轻量级锁状态下对象的Mark Word_

自旋原理非常简单，如果持有锁的线程能在很短时间内释放锁资源，那么那些等待竞争锁的线程就不需要做内核态和用户态之间的切换进入阻塞挂起状态，它们只需要等一等（自旋），等持有锁的线程释放锁后即可立即获取锁，这样就避免用户线程和内核切换的消耗。

但是，线程自旋是需要消耗`CPU`的，如果一直获取不到锁，那线程也不能一直占用`CPU`自旋做无用功，所以需要设定一个自旋等待的最大时间。`JVM`对于自旋周期的选择，`JDK 1.6`之后引入了适应性自旋锁，适应性自旋锁意味着自旋的时间不是固定的，而是由前一次在同一个锁上的自旋时间以及锁的拥有者的状态来决定的。线程如果自旋成功了，下次自旋的次数就会更多，如果自旋失败了，自旋的次数就会减少。

如果持有锁的线程执行的时间超过自旋等待的最大时间仍没有释放锁，就会导致其他争用锁的线程在最大等待时间内还是获取不到锁，自旋不会一直持续下去，这时争用线程会停止自旋进入阻塞状态，该锁膨胀为重量级锁。

#### **重量级锁状态**

重量级锁会让其他申请的线程之间进入阻塞，性能降低。重量级锁也就叫同步锁，这个锁对象`Mark Word`再次发生变化，会指向一个监视器对象，该监视器对象用集合的形式来登记和管理排队的线程。重量级锁状态下对象的`Mark Word`如下图所示，

![Desktop View](/assets/images/20260829/mark_word_heavy_lock.png){: width="600" height="300" }
_重量级锁状态下对象的Mark Word_

#### **偏向锁、轻量级锁与重量级锁的对比**

总结`synchronized`的执行过程，大致如下，
- 线程抢锁时，`JVM`首先检测内置锁对象`Mark Word`中`biased_lock`（偏向锁标识）是否设置成`1`，`lock`（锁标志位）是否为`01`，如果都满足，确认内置锁对象为可偏向状态。
- 在内置锁对象确认为可偏向状态之后，`JVM`检查`Mark Word`中线程`ID`是否为抢锁线程`ID`，如果是，就表示抢锁线程处于偏向锁状态，抢锁线程快速获得锁，开始执行临界区代码。
- 如果`Mark Word`中线程`ID`并未指向抢锁线程，就通过`CAS`操作竞争锁。如果竞争成功，就将`Mark Word`中线程`ID`设置为抢锁线程，偏向标志位设置为`1`，锁标志位设置为`01`，然后执行临界区代码，此时内置锁对象处于偏向锁状态。
- 如果`CAS`操作竞争失败，就说明发生了竞争，撤销偏向锁，进而升级为轻量级锁。
- `JVM`使用`CAS`将锁对象的`Mark Word`替换为抢锁线程的锁记录指针，如果成功，抢锁线程就获得锁。如果替换失败，就表示其他线程竞争锁，`JVM`尝试使用`CAS`自旋替换抢锁线程的锁记录指针，如果自旋成功（抢锁成功），那么锁对象依然处于轻量级锁状态。
- 如果`JVM`的`CAS`替换锁记录指针自旋失败，轻量级锁膨胀为重量级锁，后面等待锁的线程也要进入阻塞状态。

总体来说，偏向锁是在没有发生锁争用的情况下使用；一旦有了第二个线程的争用锁，偏向锁就会升级为轻量级锁；如果锁争用很激烈，轻量级锁的`CAS`自旋到达阈值后，轻量级锁就会升级为重量级锁。

三种内置锁的对比如下表所示，

|锁|优点|缺点|适用场景|
| :---- | :---- | :---- | :---- |
|偏向锁|加锁和解锁不需要额外的消耗，和执行非同步方法比仅存在纳秒级的差距|如果线程间存在锁竞争，会带来额外的锁撤销的消耗|适用于只有一个线程访问临界区场景|
|轻量级锁|竞争的线程不会阻塞，提高了程序的响应速度|抢不到锁竞争的线程使用`CAS`自旋等待，会消耗`CPU`|锁占用时间很短，吞吐量高|
|重量级锁|线程竞争不使用自旋，不会消耗`CPU`|线程阻塞，响应时间缓慢|锁占用时间较长，吞吐量最低|

### **JUC显式锁**

与`Java`内置锁不同，`JUC`显式锁提供了一种非常灵活的、使用纯`Java`语言基本的锁，这种锁的使用非常灵活，可以进行无条件的、可轮询的、定时的、可中断的锁获取和释放操作。由于`JUC`锁的加锁和解锁的方法都是通过`Java API`显式进行的，所以也叫显式锁。

使用`Java`内置锁时，不需要通过`Java`代码显式地对同步对象的监视器进行抢占和释放，这些工作由`JVM`底层完成，而且任何一个`Java`对象都能作为一个内置锁使用，所以，`Java`的对象锁使用起来非常方便。但是，`Java`内置锁的功能相对单一，不具备一些比较高级的锁功能，比如，
- 限时抢锁：在抢锁时设置超时时长，如果超时还未获得锁就放弃，不至于无限等下去。
- 可中断抢锁：在抢锁时，外部线程给抢锁线程发出一个中断信号，就能唤起等待锁的线程，并终止抢占过程。
- 多个等待队列：为锁维持多个等待队列，以便提高锁的效率。比如在生产者—消费者模式实现中，生产者和消费者共用一把锁，该锁上维持两个等待队列，即一个生产者队列和一个消费者队列。

除了以上功能问题之外，`Java`对象锁还存在性能问题。在竞争稍微激烈的情况下，`Java`对象锁会膨胀为重量级锁（基于操作系统的`Mutex Lock`实现），而重量级锁的线程阻塞和唤醒操作需要进程在内核态和用户态之间来回切换，导致其性能非常低。所以，迫切需要提供一种新的锁来提升争用激烈场景下锁的性能。

`Java`显式锁就是为了解决这些`Java`对象锁的功能问题、性能问题而生的。`JDK 5`版本引入了`Lock`接口，`Lock`是`Java`代码级别的锁。为了与`Java`对象锁相区分，`Lock`接口被称为显式锁接口，其对象实例则被称为显式锁对象。

#### **显式锁Lock接口**

`JDK 5`版本引入了`java.util.concurrent`并发包，简称为`JUC`包，里面提供了各种高并发工具类，通过此`JUC`工具包可以在`Java`代码中实现功能非常强大的多线程并发操作。所以，`Java`显式锁也被称为`JUC`显式锁。

`Lock`接口位于`java.util.concurrent.locks`包中，是`JUC`显式锁的一个抽象，`Lock`接口的主要抽象方法，如下表所示，

|方 法|描 述|
| :---- | :---- |
|`void lock()`|抢锁。成功则向下运行，若失败则阻塞抢锁线程|
|`void lockInterruptibly()`<br/>`throws InterruptedException`|可中断抢锁，当前线程在抢锁的过程中可以响应中断信号|
|`boolean tryLock()`|尝试抢锁，线程为非阻塞模式，在调用tryLock方法后立即返回。若抢锁成功则返回 true，若抢锁失败则返回 false|
|`boolean tryLock(long time, TimeUnit unit)`<br/>`throws InterruptedException`|限时抢锁，到达超时时间返回`false`。并且此限时抢锁方法也可以响应中断信号|
|`void unlock();`|释放锁|
|`Condition newCondition();`|获取与显式锁绑定的`Condition`对象，用于等待—通知方式的线程间通信|

`JUC`包中提供了一系列的显式锁实现类（如`ReentrantLock`），当然也允许应用程序提供自定义的锁实现类。

与`synchronized`关键字不同，显式锁不再作为`Java`内置特性来实现，而是作为`Java`语言可编程特性来实现。这就为多种不同功能的锁实现留下了空间，各种锁实现可能有不同的调度算法、性能特性或者锁定语义。

从`Lock`提供的接口方法可以看出，显式锁至少比`Java`内置锁多了以下优势，

可中断获取锁
使用`synchronized`关键字获取锁的时候，如果线程没有获取到被阻塞，阻塞期间该线程是不响应中断信号（`interrupt`）的；而使用`Lock.lockInterruptibly()`方法获取锁时，如果线程被中断，线程将抛出中断异常。

可非阻塞获取锁
使用`synchronized`关键字获取锁时，如果没有成功获取，线程只有被阻塞；而使用`Lock.tryLock()`方法获取锁时，如果没有获取成功，线程也不会被阻塞，而是直接返回`false`。

可限时抢锁
使用`Lock.tryLock(long time, TimeUnit unit)`方法，显式锁可以设置限定抢占锁的超时时间。而在使用`synchronized`关键字获取锁时，如果不能抢到锁，线程只能无限制阻塞。

除了以上能通过`Lock`接口直接观察出来的三点优势之外，显式锁还有不少其他的优势，稍后在介绍显式锁的种类繁多的实现类时，大家就能感觉到。

#### **可重入锁ReentrantLock**

`ReentrantLock`是`JUC`包提供的显式锁的一个基础实现类，`ReentrantLock`类实现了`Lock`接口，它拥有与`synchronized`相同的并发性和内存语义，但是拥有了限时抢占、可中断抢占等一些高级锁特性。此外，`ReentrantLock`基于内置的抽象队列同步器（`Abstract Queued Synchronized`，`AQS`）实现，在争用激烈场景下，能表现出比内置锁更佳的性能。

> 抽象队列同步器是`JUC`包同步机制的基础设施，更是`JUC`锁框架的基础。
{: .prompt-tip }

`ReentrantLock`是一个可重入的独占（或互斥）锁，其中两个修饰词的含义为，
- 可重入：表示该锁能够支持一个线程对资源的重复加锁，也就是说，一个线程可以多次进入同一个锁所同步的临界区代码块。比如，同一线程在外层函数获得锁后，在内层函数能再次获取该锁，甚至多次抢占到同一把锁。下面是一段对可重入锁进行两次抢占和释放的伪代码，具体如下，
    ```java
    lock.lock();            // 第一次获取锁
    lock.lock();            // 第二次获取锁，重新进入
    try {
        // 临界区代码块
    } finally {
        lock.unlock();      // 释放锁
        lock.unlock();      // 第二次释放锁
    }
    ```
- 独占：在同一时刻只能有一个线程获取到锁，而其他获取锁的线程只能等待，只有拥有锁的线程释放了锁后，其他的线程才能够获取锁。

使用`ReentrantLock`进行同步累加的示例如下，

```java
public class LockTest
{
    public void testReentrantLock()
    {
        // 每个线程的执行轮数
        final int TURNS = 1000;
        // 线程数
        final int THREADS = 10;

        //线程池，用于多线程模拟测试
        ExecutorService pool = Executors.newFixedThreadPool(THREADS);
        //创建一个可重入、独占锁对象

        Lock lock = new ReentrantLock();
        // 倒数门
        CountDownLatch countDownLatch = new CountDownLatch(THREADS);
        long start = System.currentTimeMillis();

        //10个线程并发执行
        for (int i = 0; i < THREADS; i++)
        {
            pool.submit(() ->
            {
                try
                {
                    //累加1000次
                    for (int j = 0; j < TURNS; j++)
                    {
                        //传入锁，执行一次累加
                        IncrementData.lockAndFastIncrease(lock);
                    }
                    log.info("本线程累加完成");
                } catch (Exception e)
                {
                    e.printStackTrace();
                }
                //线程执行完成，倒数门减少一次
                countDownLatch.countDown();
            });
        }
        try
        {
            //等待倒数门归零，所有线程结束
            countDownLatch.await();
        } catch (InterruptedException e)
        {
            e.printStackTrace();
        }
        float time = (System.currentTimeMillis() - start) / 1000F;
        //输出统计结果
        log.info("运行的时长为: " + time);
        log.info("累加结果为: " + IncrementData.sum);
    }
    //省略其他代码
}
```

显示锁可以从不同角度分为多种锁（包括乐观锁、悲观锁、公平锁、可中断锁、自旋锁等）的使用，在这些类型的锁的使用案例中，变化的部分为锁的创建代码，而不变的部分为锁的使用代码。因为`JUC`中的显式锁都实现了`Lock`接口，所以对于不同锁对象的使用代码是模板化的、套路化的。我们可以将例子中创建锁的代码（变化的部分）和使用锁的代码（不变的部分）进行分离。分离变与不变是软件设计的一个基本原则。

出于分离变与不变的设计原则，这里将临界区使用锁的代码进行了抽取和封装，形成一个可以复用的独立类——`IncrementData`累加类，具体代码如下，

```java
//封装锁的使用代码
public class IncrementData
{
    public static int sum = 0;

    public static void lockAndFastIncrease(Lock lock)
    {
        lock.lock(); //step1: 抢占锁
        try
        {
            //step2: 执行临界区代码
            sum++;
        } finally
        {
            lock.unlock(); //step3: 释放锁
        }
    }
    //省略其他代码
}
```

运行以上使用`ReentrantLock`进行累加同步的例子，其结果如下，

```
[pool‑1‑thread‑2]: 本线程累加完成
[pool‑1‑thread‑9]: 本线程累加完成
[pool‑1‑thread‑3]: 本线程累加完成
[pool‑1‑thread‑5]: 本线程累加完成
[pool‑1‑thread‑1]: 本线程累加完成
[pool‑1‑thread‑4]: 本线程累加完成
[pool‑1‑thread‑6]: 本线程累加完成
[pool‑1‑thread‑10]: 本线程累加完成
[pool‑1‑thread‑8]: 本线程累加完成
[pool‑1‑thread‑7]: 本线程累加完成
[main]: 运行的时长为: 0.126
[main]: 累加结果为: 10000
```

除了具体可重入、独占特性之外，`ReentrantLock`还支持公平锁和非公平锁两种模式。

#### **使用显示锁的三种姿势**

因为`JUC`中的显式锁都实现了`Lock`接口，所以不同类型的显式锁对象的使用方法都是模板化的、套路化的。对`lock()`、`tryLock()`、`tryLock(long time, TimeUnit unit)`这三个方法的总结如下，
- `lock()`方法用于阻塞抢锁，抢不到锁时线程会一直阻塞。
- `tryLock()`方法用于尝试抢锁，该方法有返回值，如果成功则返回`true`，如果失败（即锁已被其他线程获取）则返回`false`。此方法无论如何都会立即返回，在抢不到锁时，线程不会像使用`lock()`方法那样一直被阻塞。
- `tryLock(long time, TimeUnit unit)`方法和`tryLock()`方法是类似的，只不过这个方法在抢不到锁时会阻塞一段时间。如果在阻塞期间获取到锁立即返回`true`，超时则返回`false`。

##### **使用lock()方法抢锁**

通常情况下，大家会使用`lock()`方法进行阻塞式的锁抢占，其模板代码如下，

```java
//创建所对象，SomeLock为Lock的某个实现类，如ReentrantLock
Lock lock = new SomeLock();
lock.lock();                //step1: 抢占锁
try {
    //step2: 抢锁成功，执行临界区代码
} finally {
    lock.unlock();          //step3: 释放锁
}
```

以上抢锁模板代码有以下几个需要注意的要点，
- 释放锁操作`lock.unlock()`必须在`try‑catch`结构的`finally`块中执行，否则，如果临界区代码抛出异常，锁就有可能永远得不到释放。
- 抢占锁操作`lock.lock()`必须在`try`语句块之外，而不是放在`try`块之内。为什么呢？一是因为`lock()`方法没有声明抛出异常，所以可以不包含到`try`块中；二是因为`lock()`方法并不是一定能够抢占锁成功，如果没有抢占成功，当然也就不需要释放锁，而且在没有占有锁的情况下去释放锁，可能会导致运行时异常。
- 在抢占锁操作`lock.lock()`和`try`语句之间不要插入任何代码，避免抛出异常而无法执行释放锁操作`lock.unlock()`，导致锁无法被释放。

一段错误的抢锁代码大致如下，抢锁操作在`try`语句块之内，如果抢锁操作没有成功，也就是如果当前线程没有获取到锁，在`finally`语句块调用`unlock()`方法时就会抛出异常。

```java
Lock lock = new SomeLock();
try {
    lock.lock();        //注意：抢锁操作在try 语句块之内
    //抢锁成功，执行临界区代码
} finally {
    lock.unlock();
}
```

##### **调用tryLock()方法非阻塞抢锁**

`lock()`是阻塞式抢占，在没有抢到锁的情况下，当前线程会阻塞。如果不希望线程阻塞，可以使用`tryLock()`方法抢占锁。`tryLock()`是非阻塞抢占，在没有抢到锁的情况下，当前线程会立即返回，不会被阻塞。

调用`tryLock()`方法非阻塞抢占锁，大致的模板代码如下，

```java
//创建所对象，SomeLock为Lock的某个实现类，如ReentrantLock
Lock lock = new SomeLock();

if (lock.tryLock()) {        //step1: 尝试抢占锁
    try {
        //step2: 抢锁成功，执行临界区代码
    } finally {
        lock.unlock();       //step3: 释放锁
    }
} else {
    //step4: 抢锁失败，执行后备动作
}
```

使用`tryLock()`方法时，线程拿不到锁就立即返回，这种处理方式在实际开发中使用不多，但是其重载版本`tryLock(long time, TimeUnit unit)`方法在限时阻塞抢锁的场景中非常有用。

##### **调用 tryLock(long time, TimeUnit unit)方法抢锁**

`tryLock(long time, TimeUnit unit)`方法用于限时抢锁，该方法在抢锁时会进行一段时间的阻塞等待，其time参数代表最大的阻塞时长，其unit参数为时长的单位（如秒）。

调用`tryLock(long time, TimeUnit unit)`方法限时抢锁，其大致的代码模板如下，

```java
//创建所对象，SomeLock为Lock的某个实现类，如ReentrantLock
Lock lock = new SomeLock();
//抢锁时阻塞一段时间，如1秒
if (lock.tryLock(1, TimeUnit.SECONDS)) {  //step1: 限时阻塞抢占
    try {
        //step2: 抢锁成功，执行临界区代码
    } finally {
        lock.unlock(); //step3: 释放锁
    }
} else {
    //限时抢锁失败，执行后备动作
}
```

#### **显式锁的分类**

显式锁有很多种，从不同的角度来看，显式锁大概有以下几种分类：可重入锁与不可重入锁、悲观锁和乐观锁、公平锁和非公平锁、共享锁和独占锁、可中断锁和不可中断锁。

##### **可重入锁与不可重入锁**

从同一个线程是否可以重复占有同一个锁对象的角度来分，显式锁可以分为可重入锁与不可重入锁。

可重入锁也被称为递归锁，指的是一个线程可以多次抢占同一个锁。例如，线程`A`在进入外层函数抢占了一个`Lock`显式锁之后，当线程`A`继续进入内层函数时，如果遇到有抢占同一个`Lock`显式锁的代码，线程`A`依然可以抢到该`Lock`显式锁。

不可重入锁与可重入锁相反，指的是一个线程只能抢占一次同一个锁。例如，线程`A`在进入外层函数抢占了一个`Lock`显式锁之后，当线程`A`继续进入内层函数时，如果遇到有抢占同一个`Lock`显式锁的代码，线程`A`不可以抢到该`Lock`显式锁。除非线程`A`提前释放了该`Lock`显式锁，才能第二次抢占该锁。

`JUC`的`ReentrantLock`类是可重入锁的一个标准实现类。

##### **悲观锁和乐观锁**

从线程进入临界区前是否锁住同步资源的角度来分，显式锁可以分为悲观锁和乐观锁。

悲观锁就是悲观思想，每次去入临界区操作数据的时候都认为别的线程会修改，所以线程每次在读写数据时都会上锁，锁住同步资源，这样其他线程需要读写这个数据时就会阻塞，一直等到拿到锁。总体来说，悲观锁适用于写多读少的场景，遇到高并发写的可能性高。

`Java`的`Synchronized`重量级锁是一种悲观锁。

乐观锁是一种乐观思想，每次去拿数据的时候都认为别的线程不会修改，所以不会上锁，但是在更新的时候会去判断一下在此期间别人有没有去更新这个数据，采取在写时先读出当前版本号，然后加锁操作（比较跟上一次的版本号，如果一样就更新），如果失败就要重复读‑比较‑写的操作。总体来说，乐观锁适用于读多写少的场景，遇到高并发写的可能性低。

`Java`中的乐观锁基本都是通过`CAS`自旋操作实现的。`CAS`是一种更新原子操作，比较当前值跟传入值是否一样，是则更新，不是则失败。在争用激烈的场景下，`CAS`自旋会出现大量的空自旋，会导致乐观锁性能大大降低。

`Java`的`Synchronized`轻量级锁是一种乐观锁。另外，`JUC`中基于抽象队列同步器（`AQS`）实现的显式锁（如`ReentrantLock`）都是乐观锁。

>说明：既然在争用激烈的场景下乐观锁的性能非常低，那么为什么`JUC`的显式锁都是乐观锁？根本的原因是，`JUC`的显式锁都是基于`AQS`实现的，而`AQS`通过对队列的使用很大程度上减少了锁的争用，极大地减少了空的`CAS自`旋。所以，即使在争用激烈场景下，基于`AQS`的`JUC`乐观锁也能表现出比悲观锁更佳的性能。
{: .prompt-tip }

##### **公平锁和非公平锁**

公平锁是指不同的线程抢占锁的机会是公平的、平等的，从抢占时间上来说，先对锁进行抢占的线程一定被先满足，抢锁成功的次序体现为`FIFO`（先进先出）顺序。简单来说，公平锁就是保障了各个线程获取锁都是按照顺序来的，先到的线程先获取锁。

使用公平锁，比如线程`A`、`B`、`C`、`D`依次去获取锁，线程`A`首先获取到了锁，然后它处理完成释放锁之后，会唤醒下一个线程`B`去获取锁。后续不断重复前面的过程，线程`C`、`D`依次获取锁。

非公平锁是指不同的线程抢占锁的机会是非公平的、不平等的，从抢占时间上来说，先对锁进行抢占的线程不一定被先满足，抢锁成功的次序不会体现为`FIFO`（先进先出）顺序。

使用非公平锁，比如线程`A`、`B`、`C`、`D`依次去获取锁，假如此时持有锁的是线程`A`，然后线程`B`、`C`、`D`尝试获取锁，就会进入一个等待队列。当线程`A`释放掉锁之后，会唤醒下一个线程`B`去获取锁。在唤醒线程`B`的这个过程中，如果有别的线程`E`尝试去请求锁，那么线程`E`是可以先获取到的，这就是插队。为什么线程`E`可以插队？因为`CPU`唤醒线程`B`需要进行线程的上下文切换，这个操作需要一定的时间，线程`E`可能与线程`A`、`B`不在同一个`CPU`内核上执行，而是在其他的内核上执行，所以不需要进行线程的上下文切换。在线程`A`释放锁和线程`B`被唤醒的这段时间，锁是空闲的，其他内核上的线程`E`此时就能趁机获取非公平锁，这样做的目的主要是利用锁的空档期，提高其利用效率。

默认情况下，`ReentrantLock`实例是非公平锁，但如果在实例构造时传入了参数`true`，所得到的锁就是公平锁。另外，`ReentrantLock`的`tryLock()`方法是一个特例，一旦有线程释放了锁，正在`tryLock`的线程就能优先取到锁，即使已经有其他线程在等待队列中。

##### **可中断锁和不可中断锁**

什么是可中断锁？如果某一线程`A`正占有锁在执行临界区代码，另一线程`B`正在阻塞式抢占锁，可能由于等待时间过长，线程`B`不想等待了，想先处理其他事情，我们可以让它中断自己的阻塞等待，这种就是可中断锁。

什么是不可中断锁？一旦这个锁被其他线程占有，如果自己还想抢占，自己只能选择等待或者阻塞，直到别的线程释放这个锁，如果别的线程永远不释放锁，那么自己只能永远等下去，并且没有办法终止等待或阻塞。

简单来说，在抢锁过程中能通过某些方法去终止抢占过程，这就是可中断锁，否则就是不可中断锁。

`Java`的`synchronized`内置锁就是一个不可中断锁，而`JUC`的显式锁（如`ReentrantLock`）是一个可中断锁。

##### **独占锁和共享锁**

独占锁指的是每次只有一个线程能持有的锁。独占锁是一种悲观保守的加锁策略，它不必要地限制了读/读竞争，如果某个只读线程获取锁，那么其他的读线程都只能等待，这种情况下就限制了读操作的并发性，因为读操作并不会影响数据的一致性。

`JUC`的`ReentrantLock`类是一个标准的独占锁实现类。

共享锁允许多个线程同时获取锁，容许线程并发进入临界区。与独占锁不同，共享锁是一种乐观锁，它放宽了加锁策略，并不限制读/读竞争，允许多个执行读操作的线程同时访问共享资源。

`JUC`的`ReentrantReadWriteLock`（读写锁）类是一个共享锁实现类。使用该读写锁时，读操作可以有很多线程一起读，但是写操作只能有一个线程去写，而且在写入的时候，别的线程也不能进行读的操作。

用`ReentrantLock`锁替代`ReentrantReadWriteLock`锁虽然可以保证线程安全，但是也会浪费一部分资源，因为多个读操作并没有线程安全问题，所以在读的地方使用读锁，在写的地方使用写锁，可以提高程序执行效率。

### **CAS自旋**

#### **悲观锁存在的问题**

悲观锁可以确保无论哪个线程持有锁，都能独占式访问临界区。虽然悲观锁的逻辑非常简单，但是存在不少问题。

悲观锁总是假设会发生最坏的情况，每次线程去读取数据时，也会上锁。这样其他线程在读取数据时就会被阻塞，直到它拿到锁。传统的关系型数据库用到了很多悲观锁，比如行锁、表锁、读锁、写锁等。

悲观锁机制存在以下问题，
- 在多线程竞争下，加锁、释放锁会导致比较多的上下文切换和调度延时，引起性能问题。
- 一个线程持有锁后，会导致其他所有抢占此锁的线程挂起。
- 如果一个优先级高的线程等待一个优先级低的线程释放锁，就会导致线程的优先级倒置，从而引发性能风险。

解决以上悲观锁的这些问题的有效方式是用乐观锁去替代悲观锁。

乐观锁其实是一种思想。在使用乐观锁时，每次线程去读取数据时都认为其他线程不会修改，所以不会上锁，仅仅在更新时会判断一下其他线程有没有去更新这个数据。数据库操作中的带版本号数据更新、`JUC`包的原子类都使用了乐观锁的方式提高性能。

#### **CAS乐观锁**

乐观锁的操作主要就是两个步骤——冲突检测、数据更新。

乐观锁的一种比较典型的就是`CAS`原子操作，`JUC`强大的高并发性能是建立在`CAS`原子之上的。`CAS`操作中包含三个操作数：需要操作的内存位置（`V`）、进行比较的预期原值（`A`）和拟写入的新值（`B`）。如果内存位置`V`的值与预期原值`A`相匹配，那么处理器会自动将该位置值更新为新值`B`；否则`CPU`不做任何操作。

`CAS`操作可以非常清晰地分为两个步骤，
1. 检测位置`V`的值是否为`A`；
2. 如果是，将位置`V`更新为`B`值；否则不要更改该位置。

`CAS`的两个操作步骤其实与乐观锁操作的两个步骤是一致的，都是在冲突检测后进行数据更新。

>乐观锁是一种思想，而`CAS`是这种思想的一种实现。
{: .prompt-tip }

如果需要完成数据的最终更新，仅仅进行一次`CAS`操作是不够的，一般情况下，需要进行自旋操作，即不断地循环重试`CAS`操作直到成功，这也叫`CAS`自旋。

#### **CAS自旋**

通过`CAS`自旋，在不使用锁的情况下实现多线程之间的变量同步，也就是说，在没有线程被阻塞的情况下实现变量的同步，这叫非阻塞同步（`Non‑Blocking Synchronization`），或者说无锁同步。使用基于`CAS`自旋的乐观锁进行同步控制，属于无锁编程（`Lock Free`）的一种实践。

>如何基于`CAS`自旋实现一个简单的自旋锁？
{: .prompt-tip }

#### **不可重入的自旋锁**

自旋锁（`SpinLock`）：当一个线程在获取锁的时候，如果锁已经被其他线程获取，调用者就一直在那里循环检查该锁是否已经被释放，一直到获取到锁才会退出循环。

`CAS`自旋锁：抢锁线程不断进行`CAS`自旋操作去更新锁的`owner`（拥有者），如果更新成功，表明已经抢锁成功，退出抢锁方法。如果锁已经被其他线程获取（`owner`为其他线程），调用者就一直在那里循环进行`owner`的`CAS`更新操作，一直到成功才会退出循环。

实现一个简单版本的自旋锁——不可重入的自旋锁，具体代码如下，

```java
public class class SpinLock implements Lock
{
    /**当前锁的拥有者
     * 使用Thread作为同步状态
     */
    private AtomicReference<Thread> owner = new AtomicReference<>();
    /**
     * 抢占锁
     */
    @Override
    public void lock()
    {
        Thread t = Thread.currentThread();
        //自旋
        while (!owner.compareAndSet(null, t))
        {
            // DO nothing
            Thread.yield();//让出当前剩余的CPU时间片
        }
    }

    /**
     * 释放锁
     */
    @Override
    public void unlock()
    {
        Thread t = Thread.currentThread();
        //只有拥有者才能释放锁
        if (t == owner.get())
        {
            // 设置拥有者为空，这里不需要 compareAndSet操作
            // 因为已经通过owner做过线程检查
            owner.set(null);
        }
    }
    // 省略其他代码
}
```

仔细分析以上就可以看出，`SpinLock`是不支持重入的，即当一个线程第一次已经获取到了该锁，在锁没有被释放之前，如果又一次重新获取该锁，第二次将不能成功获取到。

#### **可重入的自旋锁**

引入一个计数器实现可重入锁，用来记录一个线程获取锁的次数。一个简单的可重入自旋锁的代码实现如下，

```java
public class ReentrantSpinLock implements Lock
{
    /**当前锁的拥有者
     * 使用拥有者Thread作为同步状态，而不是使用一个简单的整数作为同步状态
     */
    private AtomicReference<Thread> owner = new AtomicReference<>();
    /**
     * 记录一个线程重复获取锁的次数
     * 此变量为同一个线程在操作，没有必要加上volatile保障可见性和有序性
     */
    private int count = 0;

    /**
     * 抢占锁
     */
    @Override
    public void lock()
    {
        Thread t = Thread.currentThread();
        // 如果是重入，增加重入次数后返回
        if (t == owner.get())
        {
            ++count;
            return;
        }
        //自旋
        while (!owner.compareAndSet(null, t))
        {
            // DO nothing
            Thread.yield(); //让出当前剩余的CPU时间片
        }
    }

    /**
     * 释放锁
     */
    @Override
    public void unlock()
    {
        Thread t = Thread.currentThread();
        //只有拥有者才能释放锁
        if (t == owner.get())
        {
            if (count > 0)
            {
                // 如果重入的次数大于0，减少重入次数后返回
                --count;
            }
            else
            {
                // 设置拥有者为空
                //这里不需要compareAndSet，因为已经通过owner做过线程检查
                owner.set(null);
            }
        }
    }
    // 省略其他代码
}
```

自旋锁的特点：线程获取锁的时候，如果锁被其他线程持有，当前线程将循环等待，直到获取到锁。线程抢锁期间状态不会改变，一直是运行状态（`RUNNABLE`），在操作系统层面线程处于用户态。

自旋锁的问题：在争用激烈的场景下，如果某个线程持有锁的时间太长，就会导致其他空自旋的线程耗尽`CPU`资源。另外，如果大量的线程进行空自旋，还可能导致硬件层面的总线风暴。

#### **总线风暴**

`volatile`关键字原理里边，`lock`前缀指令有以下三个作用，
1. 将当前`CPU`缓存行的数据立即写回系统内存；
2. `lock`前缀指令会引起在其他`CPU`中缓存了该内存地址的数据无效；
3. `lock`前缀指令禁止指令重排。

由于在`Intel X86`平台下`CAS`的汇编指令`lock cmpxchg`也是一个`lock`前缀指令，因此`CAS`操作和`volatile`一样，也需要`CPU`进行内部通信从而保障变量的缓存一致性。

假设`Core 1`和`Core 2`可同时把某个变量加载到自己高速缓存中，当`Core1`在自己的高速缓存中修改这个位置的值时，会通过总线使`Core2`中`L1`高速缓存对应的值失效，而`Core2`一旦发现自己缓存中的值失效，就会通过总线从内存中读取最新的值，当`Core2`和`Core1`中的值再次一致时，`CPU`保障了变量的缓存一致性。

`CPU`会通过`MESI`协议保障变量的缓存一致性。为了保障缓存一致性，不同的内核需要通过总线来回通信，因而产生的流量一般称为缓存一致性流量。因为总线被设计为固定的通信能力，如果缓存一致性流量过大，总线将成为瓶颈，这就是所谓的总线风暴。

>总线风暴当然与`CPU`的架构和设计有关，并不是所有的`CPU`都会产生总线风暴。
{: .prompt-tip }

由于使用`lock`前缀指令的`Java`操作（包括`CAS`、`volatile`）恰恰会产生缓存一致性流量，当有很多线程都同时执行`lock`前缀指令操作时，在`SMP`架构的`CPU`平台上必然会导致总线风暴。

在争用激烈场景下，`Java`轻量级锁会快速膨胀为重量级锁，其本质上一是为了减少`CAS`空自旋，二是为了避免同一时间大量`CAS`操作所导致的总线风暴。

>`JUC`基于`CAS`实现的轻量级锁如何避免总线风暴呢？
使用队列对抢锁线程进行排队，最大程度上减少了`CAS`操作数量。
{: .prompt-tip }

### **CLH自旋锁**

`CLH`锁是一种基于队列（单向链表）排队的自旋锁，`Craig`、`Landin`和`Hagersten`三人一起发明的，因此被命名为`CLH`锁，也叫`CLH`队列锁。

简单的`CLH`锁可以基于单向链表实现，申请加锁的线程首先会通过`CAS`操作在单向链表的尾部增加一个节点，之后该线程只需要在其前驱节点上进行普通自旋，等待前驱节点释放锁即可。由于`CLH`锁只有在节点入队时进行一次`CAS`的操作，在节点加入队列后，抢锁线程不需要进行`CAS`自旋，只需普通自旋即可。因此，在争用激烈的场景下，`CLH`锁能大大减少的`CAS`操作的数量，以避免`CPU`的总线风暴。

>`JUC`中显式锁基于`AQS`抽象队列同步器，而`AQS`是`CLH`锁的一个变种，为了方便理解`AQS`原理，这里先看下`CLH`锁的实现和核心原理。
{: .prompt-tip }

一个`CLH`锁的简单实现版本，代码如下，

```java
public class CLHLock implements Lock
{
    /**
     * 当前节点的线程本地变量
     */
    private static ThreadLocal<Node> curNodeLocal = new ThreadLocal<>();
    /**
     * CLHLock队列的尾部指针，使用AtomicReference，方便进行CAS操作
     */
    private AtomicReference<Node> tail = new AtomicReference<>(null);

    public CLHLock()
    {
        //设置尾部节点
        tail.getAndSet(Node.EMPTYTY);
    }

    /**
     * //加锁操作：将节点添加到等待队列的尾部
     */
    @Override
    public void lock()
    {
        Node curNode = new Node(true, null);
        Node preNode = tail.get();
        //CAS自旋：将当前节点插入到队列的尾部
        while (!tail.compareAndSet(preNode, curNode))
        {
            preNode = tail.get();
        }
        //设置前驱节点
        curNode.setPrevNode(preNode);
        //自旋，监听前驱节点的locked变量，直到其值为false
        //若前驱节点的locked状态为true，则表示前一个线程还在抢占或者占有锁
        while (curNode.getPrevNode().isLocked())
        {
            //让出CPU时间片，提高性能
            Thread.yield();
        }
        //能执行到这里，说明当前线程获取到了锁
        // log.info("获取到了锁！！！");
        //将当前节点缓存在线程本地变量中，释放锁会用到
        curNodeLocal.set(curNode);
    }

    /**
     * //释放锁
     */
    @Override
    public void unlock()
    {
        Node curNode = curNodeLocal.get();
        curNode.setLocked(false);
        curNode.setPrevNode(null); //help for GC
        curNodeLocal.set(null); //方便下一次抢锁
    }

    //虚拟等待队列的节点
    @Data
    static class Node
    {
        public Node(boolean locked, Node prevNode)
        {
            this.locked = locked;
            this.prevNode = prevNode;
        }

        //true：当前线程正在抢占锁或者已经占有锁
        //false：当前线程已经释放锁，下一个线程可以占有锁了
        volatile boolean locked;
        //前一个节点，需要监听其locked字段
        Node prevNode;
        //空节点
        public static final Node EMPTYTY = new Node(false, null);
    }
    //省略其他代码
}
```

大致描述下`CLH`的抢锁和释放锁的过程，
- 抢锁线程在队列尾部加入一个节点，然后仅在前驱节点上做普通自旋，它不断轮询前一个节点状态，如果发现前一个节点释放锁，当前节点抢锁成功。
- 释放锁操作，线程从本地变量`curNodeLocal`中获取当前节点`curNode`，将其状态设置为`false`，以便的其后驱节点能获得锁。
- 线程在取当前节点`curNode`的`locked`状态设置为`false`之前，为了`GC`能回收前驱节点，需要将`curNode`前驱节点引用设置为空。另外，为了使得线程下一次抢锁不会出错，需要将线程本地变量`curNodeLocal`中的节点引用设置为空。

`CLH`有以下几个要点，
1. 初始状态队列尾部属性（`tail`）指向一个`EMPTY`节点。`tail`属性使用`AtomicReference`类型是为了使得多个线程并发操作`tail`时不会发生线程安全问题。
2. `Thread`在抢锁时会创建一个新的节点`Node`加入等待队列尾部：`tail`指向新的节点`Node`，同时新的节点`Node`的`preNode`属性指向`tail`之前指向的节点，并且以上操作通过`CAS`自旋完成，以确保操作成功。
3. `Thread`加入抢锁队列之后，会在前驱节点上自旋：循环判断前驱节点的`locked`属性是否为`false`，如果为false`就表示前驱节点释放了锁，当前线程抢占到锁。
4. `Thread`抢到锁之后，其`locked`属性一直为`true`，一直到临界区代码执行完，然后使用`unlock`方法释放锁，释放之后其`locked`属性才为`false`。

基于前面抽取出来的公共`IncrementData`累加类，编写一个`10`个线程各种累加`100000`次的累加程序，并使用`CLHLock`作为累加的同步锁。测试用例的代码如下，

```java
public class LockTest
{
    public void testCLHLockCapability()
    {
        //速度对比
        // ReentrantLock     1000000 次 0.154 秒
        // CLHLock           1000000 次 2.798 秒

        //每个线程的执行轮数
        final int TURNS = 100000;
        //线程数
        final int THREADS = 10;

        //线程池，用于多线程模拟测试
        ExecutorService pool = Executors.newFixedThreadPool(THREADS);

        Lock lock = new CLHLock();
        // Lock lock = new ReentrantLock();

        //倒数门
        CountDownLatch countDownLatch = new CountDownLatch(THREADS);
        long start = System.currentTimeMillis();
        for (int i = 0; i < THREADS; i++)
        {
            pool.submit(() ->
            {
                for (int j = 0; j < TURNS; j++)
                {
                    IncrementData.lockAndFastIncrease(lock);
                }
                log.info("本线程累加完成");
                //倒数门减少1次
                countDownLatch.countDown();
            });
        }
        try
        {
            //等待倒数门归0，所有线程结束
            countDownLatch.await();
        } catch (InterruptedException e)
        {
            e.printStackTrace();
        }
        float time = (System.currentTimeMillis() - start) / 1000F;
        //输出统计结果
        log.info("运行的时长为：" + time);
        log.info("累加结果为：" + IncrementData.sum);
    }
    //省略其他代码
}
```

运行测试用例`testCLHLockCapability`，结果如下，

```
[pool‑1‑thread‑5]: 本线程累加完成
[pool‑1‑thread‑7]: 本线程累加完成
[pool‑1‑thread‑6]: 本线程累加完成
[pool‑1‑thread‑10]: 本线程累加完成
[pool‑1‑thread‑2]: 本线程累加完成
[pool‑1‑thread‑9]: 本线程累加完成
[pool‑1‑thread‑4]: 本线程累加完成
[pool‑1‑thread‑8]: 本线程累加完成
[pool‑1‑thread‑3]: 本线程累加完成
[pool‑1‑thread‑1]: 本线程累加完成
[main]: 运行的时长为: 2.798
[main]: 累加结果为: 1000000
```

>这里的实现是一个简单版本，还存在严重的性能问题。经过对比，其性能比`JUC`的`ReentrantLock`锁差`20`倍左右。
{: .prompt-tip }

>`CLH`锁的优点是空间复杂度低。<br/>
如果有`N`个线程、`L`个锁，每个线程每次只获取一个锁，那么需要的存储空间是`O(L+N)`：`N`个线程有`N`个`Node`，`L`个锁有`L`个`Tail`。
{: .prompt-tip }

>`CLH`队列锁的一个显著缺点是，在`NUMA`架构的`CPU`平台上性能很差。<br/>
`CLH`队列锁在`NUMA`架构的`CPU`平台上，每个`CPU`内核有自己的内存，如果前驱节点在不同的`CPU`内核上，其内存位置比较远，在自旋判断前驱节点的`locked`属性时，性能将大打折扣。<br/>
不论如何，`CLH`锁在`SMP`架构的`CPU`平台上不存在这个问题，性能还是挺高的。<br/>
一种提升在`NUMA`架构下`CLH`队列锁的性能的方案是使用`MCS`队列锁。`MCS`队列锁与`CLH`队列锁的原理大致相同，具体实现在这里不做展开。
{: .prompt-tip }

### **AQS抽象同步器**

在争用激烈的场景下，使用基于`CAS`自旋实现的轻量级锁有两个问题，
1. `CAS`恶性空自旋会浪费大量的`CPU`资源；
2. 在`SMP`架构的`CPU`上会导致总线风暴。

解决`CAS`恶性空自旋的有效方式之一是以空间换时间，较为常见的方案有两种，分散操作热点、使用队列削峰，`JUC`并发包使用的是队列削峰的方案解决`CAS`的性能问题，并提供了一个基于双向队列的削峰基类——抽象基础类`AbstractQueuedSynchronizer`（抽象同步器类，`AQS`）。

`AQS`是`JUC`提供的一个用于构建锁和同步容器的基础类，建立在`CAS`原子操作和`volatile`可见性变量的基础之上，`JUC`包内的许多线程同步类都是基于`AQS`构建，比如`ReentrantLock`、`Semaphore`、`CountDownLatch`、`ReentrantReadWriteLock`、`FutureTask`等。`AQS`解决了在实现同步容器时涉及的大量细节问题。

![Desktop View](/assets/images/20260829/JUC_arch.png){: width="500" height="250"}
_AQS在JUC中的位置_

`AQS`是`CLH`队列的一个变种，主要原理和`CLH`队列差不多，`AQS`队列内部维护的是一个`FIFO`的双向链表，这种结构的特点是每个数据结构都有两个指针，分别指向直接的前驱节点和直接的后继节点。双向链表可以从任意一个节点开始很方便地访问前驱节点和后继节点。每个节点其实是由线程封装的，当线程争抢锁失败后会封装成`Node`加入到`AQS`队列中去；当获取锁的线程释放锁以后，会从队列中唤醒一个阻塞的节点（线程）。

![Desktop View](/assets/images/20260829/AQS_queue_struct.png){: width="600" height="300"}
_AQS锁的内部结构_

`AQS`出于分离变与不变的原则，基于模板模式实现。`AQS`为锁获取、锁释放的排队和出队过程提供了一系列的模板方法。由于`JUC`的显式锁种类丰富，因此`AQS`将不同锁的具体操作抽取为钩子方法，供各种锁的子类（或者其内部类）去实现。

#### **状态标志位**

`AQS`中维持了一个单一的`volatile`修饰的状态信息`state`，`AQS`使用`int`类型的`state`标示锁的状态，可以理解为锁的同步状态。

```java
//同步状态，使用volatile保证线程可见
private volatile int state;
```

以`ReentrantLock`为例，`state`初始化为`0`，表示未锁定状态。`A`线程执行该锁的`lock()`操作时，会调用`tryAcquire()`独占该锁并将`state`加`1`。此后，其他线程再`tryAcquire()`时就会失败，直到`A`线程`unlock()`到`state=0`（即释放锁）为止，其他线程才有机会获取该锁。释放锁之前，`A`线程自己是可以重复获取此锁的（`state`会累加），这就是可重入。获取多少次就要释放多么次，这样才能保证`state`能回到零态。

`AbstractQueuedSynchronizer`继承了`AbstractOwnableSynchronizer`，这个基类只有一个变量叫`exclusiveOwnerThread`，表示当前占用该锁的线程，并且提供了相应的get()和set()方法，代码如下，

```java
public abstract class AbstractOwnableSynchronizer
    implements java.io.Serializable {

    //表示当前占用该锁的线程
    private transient Thread exclusiveOwnerThread;

    //省略get/set方法
}
```

#### **队列节点类**

`AQS`是一个虚拟队列，不存在队列实例，仅存在节点之间的前后关系。节点类型通过内部类`Node`定义，核心成员如下，

```java
static final class Node {
    /**节点等待状态值1：取消状态*/
    static final int CANCELLED =  1;
    /**节点等待状态值‑1：标识后继线程处于等待状态*/
    static final int SIGNAL    = -1;
    /**节点等待状态值‑2：标识当前线程正在进行条件等待*/
    static final int CONDITION = -2;
    /**节点等待状态值‑3：标识下一次共享锁的acquireShared操作需要无条件传播*/
    static final int PROPAGATE = -3;

    //节点状态：值为SIGNAL、CANCELLED、CONDITION、PROPAGATE、0
    //普通的同步节点的初始值为0，条件等待节点的初始值为CONDITION（‑2）
    volatile int waitStatus;

    //节点所对应的线程，为抢锁线程或者条件等待线程
    volatile Thread thread;

    //前驱节点，当前节点会在前驱节点上自旋，循环检查前驱节点的waitStatus状态
    volatile Node prev;

    //后继节点
    volatile Node next;

    //如果当前Node不是普通节点而是条件等待节点，则节点处于某个条件的等待队列上
    //此属性指向下一个条件等待节点，即其条件队列上的后继节点
    Node nextWaiter;
    ...
}
```

1. waitStatus属性，每个节点与等待线程关联，每个节点维护一个状态waitStatus，waitStatus的各种值以常量的形式进行定义。
2. thread成员，Node的thread成员用来存放进入AQS队列中的线程引用；Node的nextWaiter成员用来指向自己的后继等待节点，此成员只有线程处于条件等待队列中的时候使用。
3. 抢占类型常量标识，Node节点还定义了两个抢占类型常量标识：SHARED和EXCLUSIVE，SHARED表示线程是因为获取共享资源时阻塞而被添加到队列中的；EXCLUSIVE表示线程因为获取独占资源时阻塞而被添加到队列中的。代码如下，
    ```java
    static final class Node {
        //标识节点在抢占共享锁
        static final Node SHARED = new Node();
        //标识节点在抢占独占锁
        static final Node EXCLUSIVE = null;
        ...
    }
    ```

#### **FIFO双向同步队列**

`AQS`的内部队列是`CLH`队列的变种，每当线程通过`AQS`获取锁失败时，线程将被封装成一个`Node`节点，通过`CAS`原子操作插入队列尾部。当有线程释放锁时，`AQS`会尝试让队首的后继节点占用锁。

`AQS`是一个通过内置的`FIFO`双向队列来完成线程的排队工作，内部通过节点`head`和`tail`记录队首和队尾元素，元素的节点类型为`Node`类型，代码如下，

```java
/*首节点的引用*/
private transient volatile Node head;
/*尾节点的引用*/
private transient volatile Node tail;
```

`AQS`的队首节点和队尾节点都是懒加载的，即在需要的时候才真正创建。只有在线程竞争失败的情况下，有新线程加入同步队列时，`AQS`才创建一个`head`节点。`head`节点只能被`setHead()`方法修改，并且节点的`waitStatus`不能为`CANCELLED`。队尾节点只在有新线程阻塞时才被创建。

#### **JUC显式锁与AQS的关系**

`AQS`是`java.util.concurrent`包的一个同步器，它实现了锁的基本抽象功能，支持独占锁与共享锁两种方式。该类使用模板模式来实现的，成为构建锁和同步器的框架，使用该类可以简单且高效地构造出应用广泛的同步器（或者等待队列）。

`java.util.concurrent.locks`包中的显式锁如`ReentrantLock`、`ReentrantReadWriteLock`，线程同步工具如`Semaphore`，异步回调工具如`FutureTask`等，内部都使用了`AQS`作为等待队列。通过开发工具进行`AQS`的子类导航会发现大量的`AQS`子类以内部类的形式使用，也能继承`AQS`类去实现自己需求的同步器（或锁）。

以`ReentrantLock`为例，看下`ReentrantLock`与`AQS`的组合关系。

`ReentrantLock`是一个可重入的互斥锁，又称为可重入独占锁。`ReentrantLock`锁在同一个时间点只能被一个线程锁持有，而可重入的意思是，`ReentrantLock`锁可以被单个线程多次获取。

`ReentrantLock`把所有`Lock`接口的操作都委派到一个`Sync`类上，该类继承了`AbstractQueuedSynchronizer`，

```java
static abstract class Sync extends AbstractQueuedSynchronizer {...}
```

`ReentrantLock`为了支持公平锁和非公平锁两种模式，为`Sync`又定义了两个子类，如下，

```java
final static class NonfairSync extends Sync {...}
final static class FairSync extends Sync {...}
```

`NonfairSync`为非公平（或者不公平）同步器，`FairSync`为公平同步器。`ReentrantLock`提供了两个构造器，如下，

```java
public ReentrantLock() {                //默认的构造器
    sync = new NonfairSync();           //内部使用非公平同步器
}

public ReentrantLock(boolean fair) {    //true 为公平锁，否则为非公平锁
    sync = fair ? new FairSync() : new NonfairSync();
}
```

`ReentrantLock`的默认构造器（无参数构造器）被初始化为一个`NonfairSync`对象，即使用非公平同步器，所以，默认情况下`ReentrantLock`为非公平锁。带参数的构造器可以根据`fair`参数的值具体指定`ReentrantLock`的内部同步器使用`FairSync`还是`NonfairSync`。

从`ReentrantLock`的`lock()`和`unlock()`的源码可以看到，它们只是分别调用了`sync`对象的`lock()`和`release()`方法。

```java
public void lock() {                    //抢占显式锁
    sync.lock();
}

public void unlock() {                  //释放显式锁
    sync.release(1);
}
```

通过上面的委托代码可以看出，`ReentrantLock`的显式锁操作是委托（或委派）给一个`Sync`内部类的实例完成的。而`Sync`内部类只是`AQS`的一个子类，所以本质上`ReentrantLock`的显式锁操作是委托（或委派）给`AQS`完成的。一个`ReentrantLock`对象的内部一定有一个`AQS`类型的组合实例，二者之间是组合关系。

![Desktop View](/assets/images/20260829/reentrantlock_class_struct.png){: width="600" height="300"}
_ReentrantLock的内部结构_

#### **AQS中的模板模式**

模板模式将不变的部分封装在基类的骨架方法中，而将变化的部分通过钩子方法进行封装，交给子类去提供具体的实现，在一定程度上优美地阐述了分离变与不变这一软件设计原则。

`AQS`同步器是基于模板模式设计的，并且是模板模式经典的一个运用。如果需要自定义同步器，一般的方法是继承`AQS`，并重写指定方法（钩子方法），按照自己定义的规则对`state`（锁的状态信息）进行获取与释放；将`AQS`组合在自定义同步组件的实现中，自定义同步器去调用`AQS`的模板方法，而这些模板方法会调用重写的钩子方法。

`AQS`定义了两种资源共享方式，
- `Exclusive`（独享锁）：只有一个线程能占有锁资源，如`ReentrantLock`。独享锁又可分为公平锁和非公平锁。
- `Share`(共享锁)：多个线程可同时占有锁资源，如`Semaphore`、`CountDownLatch`、`CyclicBarrier`、`ReadWriteLock`的`Read`锁。

`AQS`为不同的资源共享方式提供了不同的模板流程，包括共享锁、独享锁模板流程。这些模板流程完成了具体线程进出等待队列的基础（如获取资源失败入队/唤醒出队等）、通用逻辑。基于基础、通用逻辑，`AQS`提供一种实现阻塞锁和依赖`FIFO`等待队列的同步器框架，`AQS`模板为`ReentrantLock`、`CountDownLatch`、`Semaphore`提供了优秀的解决方案。

自定义的同步器只需要实现共享资源`state`的获取与释放方式即可，这些逻辑都编写在钩子方法中。无论是共享锁还是独享锁，`AQS`在执行模板流程时会回调自定义的钩子方法。

自定义同步器时，`AQS`中需要重写的钩子方法如下，
- `tryAcquire(int)`：独占锁钩子，尝试获取资源。若成功则返回`true`，若失败则返回`false`。
- `tryRelease(int)`：独占锁钩子，尝试释放资源。若成功则返回`true`，若失败则返回false。
- `tryAcquireShared(int)`：共享锁钩子，尝试获取资源，负数表示失败；`0`表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源。
- `tryReleaseShared(int)`：共享锁钩子，尝试释放资源。若成功则返回`true`，若失败则返回`false`。
- `isHeldExclusively()`：独占锁钩子，判断该线程是否正在独占资源。只有用到`condition`条件队列时才需要去实现它。

以上钩子方法的默认实现会抛出`UnsupportedOperationException`异常。除了这些钩子方法外，`AQS`类中的其他方法都是`final`类型的方法，所以无法被其他类继承，只有这几个方法可以被其他类继承。

#### **简单的独占锁的实现**

`SimpleMockLock`是一个基于`AQS`的、简单的非公平独占锁实现，代码如下，

```java
public class SimpleMockLock implements Lock
{
    //同步器实例
    private final Sync sync = new Sync();

    // 自定义的内部类：同步器
    // 直接使用 AbstractQueuedSynchronizer.state 值表示锁的状态
    // AbstractQueuedSynchronizer.state=1 表示锁没有被占用
    // AbstractQueuedSynchronizer.state=0 表示锁已经被占用
    private static class Sync extends AbstractQueuedSynchronizer
    {
        //钩子方法
        protected boolean tryAcquire(int arg)
        {
            //CAS更新状态值为1
            if (compareAndSetState(0, 1))
            {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }

        //钩子方法
        protected boolean tryRelease(int arg)
        {
            //如果当前线程不是占用锁的线程
            if (Thread.currentThread() != getExclusiveOwnerThread())
            {
                //抛出非法状态的异常
                throw new IllegalMonitorStateException();
            }
            //如果锁的状态为没有占用
            if (getState() == 0)
            {
                //抛出非法状态的异常
                throw new IllegalMonitorStateException();
            }
            //接下来不需要使用CAS操作，因为下面的操作不存在并发场景
            setExclusiveOwnerThread(null);
            //设置状态
            setState(0);
            return true;
        }
    }

    // 显式锁的抢占方法
    // 抢锁：将节点添加到等待队列的尾部
    @Override
    public void lock()
    {
        // 委托给同步器的acquire()抢占方法
        // 开启同步器的抢锁流程，将节点添加到等待队列的尾部
        sync.acquire(1);
    }

    // 显式锁的释放方法
    @Override
    public void unlock()
    {
        // 委托给同步器的release()释放方法
        sync.release(1);
    }
    // 省略其他未实现的方法
}
```

`SimpleMockLock`仅仅实现了`Lock`接口的以下两种方法，
- `lock()`方法：完成显式锁的抢占；
- `unlock()`方法：完成显式锁的释放。

`SimpleMockLock`的锁抢占和锁释放是委托给`Sync`实例的`acquire()`方法和`release()`方法完成的。`SimpleMockLock`的内部类`Sync`继承了`AQS`类，实际上`acquire()`、`release()`是`AQS`的两个模板方法。在抢占锁时，`AQS`的模板方法`acquire()`会调用`tryAcquire(int arg)`钩子方法；在释放锁时，`AQS`的模板方法`release()`会调用`tryRelease(int arg)`钩子方法。

内部类`Sync`继承`AQS`类时提供了以下两个钩子方法的实现，
1. `protected boolean tryAcquire(int arg)`：抢占锁的钩子实现。此方法将锁的状态设置为`1`，表示互斥锁已经被占用，并保存当前线程；
2. `protected boolean tryRelease(int arg)`：释放锁的钩子实现。此方法将锁的状态设置为`0`，表示互斥锁已经被释放。

基于前面抽取出来的公共`IncrementData`累加类编写一个让`10`个线程各累加`1000`次的程序，并使用`SimpleMockLock`作为累加的同步锁。`SimpleMockLock`的测试用例如下，

```java
public class LockTest
{
    public void testMockLock()
    {
        // 每个线程的执行轮数
        final int TURNS = 1000;
        // 线程数
        final int THREADS = 10;

        // 线程池，用于多线程模拟测试
        ExecutorService pool = Executors.newFixedThreadPool(THREADS);
        // 自定义的独占锁
        Lock lock = new SimpleMockLock();
        // 倒数门
        CountDownLatch countDownLatch = new CountDownLatch(THREADS);
        long start = System.currentTimeMillis();

        // 10个线程并发执行
        for (int i = 0; i < THREADS; i++)
        {
            pool.submit(() ->
            {
                try
                {
                    // 累加 1000 次
                    for (int j = 0; j < TURNS; j++)
                    {
                        // 传入锁，执行一次累加
                        IncrementData.lockAndFastIncrease(lock);
                    }
                    log.info("本线程累加完成");
                } catch (Exception e) {
                    e.printStackTrace();
                }
                // 线程执行完成，倒数门减少一次
                countDownLatch.countDown();
            });
        }
        // 省略等待并发执行完成、结果输出的代码
    }
}
```

#### **AQS锁抢占过程**

基于`SimpleMockLock`公平独占锁的抢占过程详细看下`AQS`锁抢占的原理。

![Desktop View](/assets/images/20260829/mock_lock_acquire_flow.png){: width="600" height="300"}
_AQS锁抢占过程_

流程的第一步，显式锁的`lock()`方法会去调用同步器基类`AQS`的模板方法`acquire(arg)`。

##### **AQS模板方法：acquire(arg)**

`acquire`是`AQS`封装好的获取资源的公共入口，它是`AQS`提供的利用独占方式获取资源的方法，源码如下，

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

通过源码可以发现，`acquire(arg)`至少执行一次`tryAcquire(arg)`钩子方法。`tryAcquire(arg)`方法默认是抛出一个异常，具体的获取独占资源`state`的逻辑需要钩子方法去实现。

在模板方法`acquire`中，若调用`tryAcquire(arg)`尝试成功，则`acquire()`将直接返回，表示已经抢到锁；若不成功，则将线程加入等待队列。

##### **钩子实现：tryAcquire(arg)**

`SimpleMockLock`的`tryAcquire()`的流程是，`CAS`操作`state`字段，将其值从`0`改为`1`，若成功，则表示锁未被占用，可成功占用，并且返回`true`；若失败，则获取锁失败，返回`false`。

`SimpleMockLock`的实现非常简单，是不可以重入的。如果是可重入的锁，在重复抢锁时会累计`state`字段值，表示重入锁的次数，具体可参考`ReentrantLock`源码。

##### **直接入队：addWaiter**

在`acquire`模板方法中，如果钩子方法`tryAcquire`尝试获取同步状态失败，则构造同步节点（独占式节点模式为`Node.EXCLUSIVE`），通过`addWaiter(Node node, int args)`方法将该节点加入到同步队列的队尾。

```java
private Node addWaiter(Node mode) {
    //创建新节点
    Node node = new Node(Thread.currentThread(), mode);
    // 加入队列尾部，将目前的队列tail作为自己的前驱节点pred
    Node pred = tail;
    // 队列不为空的时候
    if (pred != null) {
        node.prev = pred;
        // 先尝试通过AQS方式修改尾节点为最新的节点
        // 如果修改成功，将节点加入到队列的尾部
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    //第一次尝试添加尾部失败，意味着有并发抢锁发生，需要进行自旋
    enq(node);
    return node;
}
```

在`addWaiter()`方法中，首先需要构造一个`Node`对象，构造`Node`对象用到的两个参数如下，
1. 当前线程
    - 构造`Node`对象时，将通过`Thread.currentThread()`获取到当前线程作为第一个参数，该线程会被赋值给`Node`对象的`thread`成员属性，相当于将线程与`Node`节点进行绑定。在后续轮到此`Node`节点去占用锁时，就需要其`thread`属性获得需要唤醒的线程。
2. `Node`共享类型
    - `mode`表示`Node`类型，用于标识新节点是独占还是共享去抢占锁。`mode`虽然为`Node`类型，但是仅仅起到类型标识的作用。`mode`可能的值有两个，以常量的形式定义在`Node`类中，如下，
    ```java
    static final class Node {
        /** 常量标识：标识当前的队列节点类型为共享型抢占 */
        static final Node SHARED = new Node();
        /** 常量标识：标识当前的队列节点类型为独占型抢占 */
        static final Node EXCLUSIVE = null;
        //省略其他代码
    }
    ```

##### **自旋入队：enq**

`addWaiter()`第一次尝试在尾部添加节点失败，意味着有并发抢锁发生，需要进行自旋。`enq()`方法通过`CAS`自旋将节点的添加到队列尾部。

```java
/**
 * 这里进行了循环，如果此时存在tail，就执行添加队尾的操作
 * 如果依然不存在，就把当前线程作为head节点
 * 插入节点后，调用acquireQueued()进行阻塞
 */
private Node enq(final Node node) {
    for (;;) { // 自旋入队
        Node t = tail;
        if (t == null) {
            //队列为空，初始化队尾节点和队首节点为新节点
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            //队列不为空，将新节点插入队列尾部
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}

/**
 * CAS操作head指针，仅仅被enq()调用
 */
private final boolean compareAndSetHead(Node update) {
    return unsafe.compareAndSwapObject(this, headOffset, null, update);
}

/**
 * CAS操作tail指针，仅仅被enq()调用
 */
private final boolean compareAndSetTail(Node expect, Node update) {
    return unsafe.compareAndSwapObject(this, tailOffset, expect, update);
}
```

##### **自旋抢占：acquireQueued()**

在节点入队之后，启动自旋抢锁的流程。

`acquireQueued()`方法的主要逻辑：当前`Node`节点线程在死循环中不断获取同步状态，并且不断在前驱节点上自旋，只有当前驱节点是队首节点才能尝试获取锁，原因是，
1. 队首节点是成功获取同步状态（锁）的节点，而队首节点的线程释放了同步状态以后，将会唤醒其后驱节点，后驱节点的线程被唤醒后要检查自己的前驱节点是否为队首节点；
2. 维护同步队列的`FIFO`原则，节点进入同步队列之后，就进入了一个自旋的过程，每个节点都在不断地执行`for`死循环。

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        // 在前驱节点上自旋
        // 自旋检查当前节点的前驱节点是否为队首节点，才能获取锁
        for (;;) {
            // 获取节点的前驱节点
            final Node p = node.predecessor();
            // 节点中的线程循环的检查自己的前驱节点是否为head节点（前驱节点是队首节点）
            // 只有前驱节点是head时，进一步调用子类的tryAcquire(…)实现（通过子类的tryAcquire钩子实现抢占成功）
            if (p == head && tryAcquire(arg)) {
                // tryAcquire成功后，将当前节点设置为队首节点，移除之前的队首节点
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            // 检查前一个节点的状态，预判当前获取锁失败的线程是否要挂起
            // 如果需要挂起
            // 调用parkAndCheckInterrupt方法挂起当前线程，直到被唤醒
            if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                interrupted = true; // 若两个操作都是true，则为true
        }
    } finally {
        //如果等待过程中没有成功获取资源（如timeout，或者可中断的情况下被中断了）
        //那么取消节点在队列中的等待
        if (failed)
            //取消请求，将当前节点从队列中移除
            cancelAcquire(node);
    }
}
```

为了不浪费资源，`acquireQueued()`自旋过程中会阻塞线程，等待前驱节点唤醒后才启动循环。如果成功就返回，否则执行`shouldParkAfterFailedAcquire()`、`parkAndCheckInterrupt()`来达到阻塞效果。

调用`acquireQueued()`方法的线程一定是`node`所绑定的线程（由它的`thread`属性所引用），该线程也是最开始调用`lock()`方法抢锁的那个线程，在`acquireQueued()`的死循环中，该线程可能重复进行阻塞和被唤醒。

`AQS`队列上每一个节点所绑定的线程在抢锁过程中都会自旋，即执行`acquireQueued()`方法的死循环，也就是说，`AQS`队列上每个节点的线程都不断自旋。

如果队首节点获取了锁，那么该节点绑定的线程会终止`acquireQueued()`自旋，线程会去执行临界区代码。此时，其余的节点处于自旋状态，处于自旋状态的线程当然也不会执行无效的空循环而导致`CPU`资源浪费，而是被挂起（`Park`）进入阻塞状态。`AQS`队列的节点自旋不像`CLH`节点那样在空自旋而耗费资源。

##### **挂起预判：shouldParkAfterFailedAcquire**

`acquireQueued()`自旋在阻塞自己的线程之前会进行挂起预判。`shouldParkAfterFailedAcquire()`方法的主要功能是，找到当前节点的有效前驱节点（是指有效节点不是`CANCELLED`类型的节点），并且将有效前驱节点的状态设置为`SIGNAL`，之后返回`true`代表当前线程可以马上被阻塞了。具体可以分为三种情况，
1. 如果前驱节点的状态为`‑1`（`SIGNAL`），说明前驱的等待标志已设好，返回`true`表示设置完毕。
2. 如果前驱节点的状态为`1`（`CANCELLED`），说明前驱节点本身不再等待了，需要跨越这些节点，然后找到一个有效节点，再把当前节点和这个有效节点的唤醒关系建立好；调整前驱节点的`next`指针为自己。
3. 如果是其他情况：`‑3`（`PROPAGATE`、共享锁等待）、`‑2`（`CONDITION`、条件等待）、`0`（初始状态），那么通过`CAS`尝试设置前驱节点为`SIGNAL`，表示只要前驱释放锁，当前节点就可以抢占锁了。

`shouldParkAfterFailedAcquire`的源码如下，

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;        // 获得前驱节点的状态
    if (ws == Node.SIGNAL)           //如果前驱节点状态为SIGNAL（值为‑1）就直接返回
        return true;
    if (ws > 0) {                    // 前驱节点以及取消CANCELLED (1)
        do {
            // 不断地循环，找到有效前驱节点，即非CANCELLED（值为1）类型节点
            //将pred记录前驱的前驱
            pred = pred.prev;
            //调整当前节点的prev指针，保持为前驱的前驱
            node.prev = pred;
        } while (pred.waitStatus > 0);
        //调整前驱节点的next指针
        pred.next = node;
    } else {
        //如果前驱状态不是CANCELLED，也不是SIGNAL，就设置为SIGNAL
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        //设置前驱状态之后，此方法返回值还是为false，表示线程不可用，被阻塞
    }
    return false;
}
```

在独占锁的场景中，此方法`shouldParkAfterFailedAcquire()`是在`acquireQueued()`方法的死循环中被调用的，由于此方法返回`false`时`acquireQueued()`不会阻塞当前线程，只有此方法返回`true`时当前线程才阻塞。因此在一般情况下，此方法至少需执行两次，当前线程才会被阻塞。

在第一次进入此方法时，首先会进入后一个`if`判断的`else`分支，通过`CAS`设置`pred`前驱的`waitStatus`为`SIGNAL`，然后返回`false`。

此方法返回`false`之后，获取独占锁的`acquireQueued()`方法会继续进行`for`循环去抢锁，
1. 假设`node`的前驱节点是队首节点，`tryAcquire()`抢锁成功，则获取到锁。
2. 假设`node`的前驱节点仍然不是队首节点，或`tryAcquire()`抢锁失败，仍会再次调用此方法。

第二次进入此方法时，由于上一次进入时已经将`pred.waitStatus`设置为`‑1`（`SIGNAL`）了，因此这次会进入第一个判断条件，直接返回`true`，表示应该调用`parkAndCheckInterrupt`阻塞当前线程了，等待前一个节点执行完成之后唤醒。
1. `waitStatus == ‑3`
    - 什么时候遇到前驱节点状态`waitStatus`等于`‑3`（`PROPAGATE`）的场景？
    - `PROPAGATE`只能在使用共享锁的时候出现，并且只可能设置在`head`上。所以，对于非队尾节点，如果它的状态为`0`或`PROPAGATE`，那么它肯定是`head`。当等待队列中有多个节点时，如果`head`的状态为`0`或`PROPAGATE`，说明`head`处于一种中间状态，且此时有线程刚才释放锁了。而对于抢锁线程来说，如果检测到这种状态，说明再次执行`acquire()`是极有可能获得锁的。
2. `waitStatus > 0`
    - 什么时候会遇到前驱节点的状态waitStatus大于0的场景呢？
    - 当pred前驱节点的抢锁请求被取消后期状态为CANCELLED（值为1）时，当前节点（如果被唤醒）就会循环移除所有被取消的前驱节点，直到找到未被取消的前驱。在移除所有被取消的前驱节点后，此方法将返回false，再一次去执行acquireQueued()的自旋抢占。
3. `waitStatus == 0`
    - 什么时候遇到前驱节点状态`waitStatus == 0`（初始状态）的场景？
    - 分为两种情况，
        - `node`节点刚成为新队尾，但还没有将旧队尾的状态设置为`SIGNAL`；
        - `node`节点的前驱节点为`head`。

>前驱节点为`waitStatus==0`的情况是最常见的。比如现在`AQS`的等待队列中有很多节点正在等待，当前线程刚执行完毕`addWaiter`（节点刚成为新队尾），然后开始执行获取锁的死循环（独占锁对应的是`acquireQueued()`里的死循环，共享锁对应的是`doAcquireShared()`里的死循环），此时节点的前驱（也就是旧队尾的状态）肯定还是`0`（也就是默认初始化的值），然后死循环执行两次，第一次执行`shouldParkAfterFailedAcquire()`自然会检测到前驱状态为`0`，然后将`0`设置为`SIGNAL`；第二次执行`shouldParkAfterFailedAcquire()`，由于前驱节点为`SIGNAL`，当前线程直接返回`true`，去执行自我阻塞。
{: .prompt-tip }

##### **线程挂起：parkAndCheckInterrupt()**

`parkAndCheckInterrupt()`主要任务是暂停当前线程，如下，

```java
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);         // 调用park()使线程进入waiting状态
    return Thread.interrupted();    // 如果被唤醒，查看自己是否已经被中断
}
```

`AbstractQueuedSynchronizer`会把所有的等待线程构成一个阻塞等待队列，当一个线程执行完`lock.unlock()`时，会激活其后驱节点，通过调用`LockSupport.unpark(postThread)`完成后继线程的唤醒。

#### **AQS中节点的入队和出队**

理解`AQS`的原理，一个比较重要的点在于掌握节点的入队和出队。

##### **节点的自旋入队**

节点在第一次入队失败后，就会开始自旋入队，分为以下两种情况，
1. 如果`AQS`的队列非空，新节点通过`CAS`插入队列尾部，并且是通过`CAS`方式插入，插入之后`AQS`的`tail`将指向新的尾节点；
2. 如果`AQS`的队列为空，新节点入队时，`AQS`通过`CAS`方法将新节点设置为队首节点，并且将`tail`指针指向新节点。然后自旋，进入`CAS`插入操作，直到插入成功，自旋才结束。

节点的入队的代码在`enq()`方法中，参见[自旋入队：enq](https://shouyuanman.github.io/posts/java-thread-safe/#aqs%E9%94%81%E6%8A%A2%E5%8D%A0%E8%BF%87%E7%A8%8B)

>队列初始化创建了一个空的队首节点，这个空的队首节点没有对应的线程，只占用一个位置，等到后面的节点抢到锁，这个节点就被移除。
{: .prompt-tip }

>梳理一下节点进入`AQS`的时机，下面梳理了两个时机和三种细分场景。
- **时机一：**在模板方法`acquire()`中，如果调用`tryAcquire(arg)`尝试成功，`acquire()`将直接返回，表示已经抢到锁；如果不成功，则开始将线程加入等待队列。这里分为三种场景，
    1. 模板方法`acquire(arg)`通过`addWaiter(Node node, int args)`方法，尝试将该节点加入到同步队列的队尾，在存在竞争的场景时一般会成功。当然，如果加入失败，或者同步队列为空，就开始调用`enq(final Node node)`自旋入队。
    2. `enq()`方法通过`CAS`自旋将新节点插入队列尾部。具体来说，如果`AQS`的队列非空，新节点入队的插入位置在队列的尾部，并且是通过`CAS`方式插入的，插入之后`AQS`的`tail`将指向新节点，新节点作为尾节点。
    3. `enq()`方法初始化`AQS`队列再执行`CAS`自旋。如果`AQS`的队列为空，新节点入队时首先进行队列初始化，`AQS`通过`CAS`方法创建队首节点，并且将`tail`指针指向队首节点。然后自旋，进入`CAS`自旋插入操作，直到插入成功，自旋才结束。
- **时机二：**`Condition`等待队列上的节点被`signal()`唤醒，会通过`enq(final Node node)`自旋入队，插入`AQS`的尾部。
{: .prompt-tip }

##### **节点的出队**

节点出队的算法在`acquireQueued()`方法中，参见[自旋抢占：acquireQueued()](https://shouyuanman.github.io/posts/java-thread-safe/#aqs%E9%94%81%E6%8A%A2%E5%8D%A0%E8%BF%87%E7%A8%8B)。

`acquireQueued()`方法通过不断在前驱节点上自旋（`for`死循环），如果前驱节点是队首节点，并且当前线程使用钩子方法`tryAcquire(arg)`获得了锁，则移除队首节点，将当前节点设置为队首节点。

节点加入到队列尾部后，如果其前驱节点就不是队首节点，通常情况下，该新节点所绑定的线程会被无限期阻塞，而不会去执行无效循环，从而导致`CPU`资源的浪费。

>被无限期阻塞的抢锁线程，是什么时候被唤醒的？<br/>
对于公平锁而言，队首节点就是占用锁的节点，在释放锁时，将会唤醒其后驱节点所绑定的线程。后驱节点的线程被唤醒后会重新执行以上`acquireQueued()`的自旋（`for`死循环）抢锁逻辑，检查自己的前驱节点是否为队首节点，如果是，在抢锁成功之后会移除旧的队首节点。
{: .prompt-tip }

>`AQS`释放锁时是如何唤醒后继线程的呢？<br/>
无效节点的出队操作是在唤醒后驱节点的线程之后，其后驱节点的线程在抢锁过程中完成的。
{: .prompt-tip }

`AQS`释放锁的核心代码如下，

```java
public final boolean release(long arg) {
    if (tryRelease(arg)) { // 释放锁的钩子实现
        Node h = head; //队列的队首节点
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h); //唤醒后继线程
        return true;
    }
    return false;
}

private void unparkSuccessor(Node node) {
    // 省略不相关代码
    Node s = node.next; //后驱节点
    // 省略不相关代码
    if (s != null)
        LockSupport.unpark(s.thread); //唤醒后驱的线程
}
```

#### **AQS锁释放过程**

`SimpleMockLock`的`unlock()`方法被调用时，会调用`AQS`的`release(...)`的模板方法。

![Desktop View](/assets/images/20260829/mock_lock_release_flow.png){: width="600" height="300"}
_AQS锁释放过程_

`AQS`的`release(...)`模板方法代码如下，

```java
public final boolean release(long arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

代码逻辑比较简单，如果同步状态的钩子方法执行成功（`tryRelease`返回`true`），就会执行`if`块中的代码，当`head`指向的队首节点不为`null`，并且该节点的状态值不为`0`时才会执行`unparkSuccessor()`方法。

钩子方法`tryRelease()`方法尝试释放当前线程持有的资源，需要子类根据具体业务去实现，核心逻辑是设置同步状态`state`的值为`0`，方便后继节点执行抢占，具体代码参见[SimpleMockLock实现](https://shouyuanman.github.io/posts/java-thread-safe/#%E7%AE%80%E5%8D%95%E7%9A%84%E7%8B%AC%E5%8D%A0%E9%94%81%E7%9A%84%E5%AE%9E%E7%8E%B0)。

`release()`钩子执行了`tryRelease()`钩子成功之后，使用`unparkSuccessor()`唤醒后继节点，代码如下，

```java
private void unparkSuccessor(Node node) {
    int ws = node.waitStatus; // 获得节点状态，释放锁的节点，也就是队首节点
    //CANCELLED (1)、SIGNAL (-1)、CONDITION (-2)、PROPAGATE (-3)
    //如果队首节点状态小于0，则将其置为0，表示初始状态
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    Node s = node.next; // 找到后面的一个节点
    if (s == null || s.waitStatus > 0) {
        // 如果新节点已经被取消CANCELLED (1)
        s = null;
        //从队列尾部开始，往前去找最前面的一个waitStatus小于0的节点
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)  s = t;
    }
    //唤醒后继节点对应的线程
    if (s != null)
        LockSupport.unpark(s.thread);
}
```

`unparkSuccessor()`唤醒后继节点的线程后，后继节点的线程重新执行方法`acquireQueued()`中的自旋抢占逻辑。

>当`AQS`队首节点释放锁之后，队首节点的状态变成初始状态，此节点理论上需要从队列中移除，但此时该无效节点并没有立即被移除，`unparkSuccessor()`方法并没有立即从队列中删除该无效节点，仅仅唤醒了后继节点的线程，重启了后继节点的自旋抢锁。
{: .prompt-tip }

#### **AQS条件队列**

`Condition`是`JUC`用来替代传统`Object`的`wait()/notify()`线程间通信协作机制的新组件，前者实现线程间协作更加高效。

##### **Condition基本原理**

`Condition`与`Object`的`wait()/notify()`作用是相似的，都是使得一个线程等待某个条件（`Condition`），只有当该条件具备`signal()`或者`signalAll()`方法被调用时等待线程才会被唤醒，从而重新争夺锁。不同的是，`Object`的`wait()/notify()`由`JVM`底层实现，而`Condition`接口与实现类完全使用`Java`代码实现。

当需要进行线程间通信时，建议结合使用`ReentrantLock`与`Condition`，通过`Condition`的`await()`和`signal()`方法进行线程间的阻塞与唤醒。

`ConditionObject`类是实现条件队列的关键，每个`ConditionObject`对象都维护一个单独的条件等待队列。每个`ConditionObject`对应一个条件队列，它记录该队列的队首节点和尾节点。

```java
public class ConditionObject implements Condition, java.io.Serializable {
    //记录该队列的队首节点
    private transient Node firstWaiter;
    //记录该队列的尾节点
    private transient Node lastWaiter;
}
```

在一个显式锁上，我们可以创建多个等待任务队列，这点和内置锁不同，`Java`内置锁上只有唯一的一个等待队列。比如，我们可以使用`newCondition()`创建两个等待队列，具体如下，

```java
private Lock lock = new ReentrantLock();
//创建第一个等待队列
private Condition firstCond = lock.newCondition();
//创建第二个等待队列
private Condition secondCond = lock.newCondition();
```

`Condition`条件队列与`AQS`同步队列的关系如下，

![Desktop View](/assets/images/20260829/AQS_condition_queue.png){: width="600" height="300"}
_Condition条件队列与AQS同步队列的关系_

`Condition`条件队列是单向的，而`AQS`同步队列是双向的，`AQS`节点会有前驱指针。一个`AQS`实例可以有多个条件队列，是聚合关系；但是一个`AQS`实例只有一个同步队列，是逻辑上的组合关系。

##### **await()等待方法**

当线程调用`await()`方法时，说明当前线程的节点为当前`AQS`队列的队首节点，正好处于占有锁的状态，`await()`方法需要把该线程从`AQS`队列挪到`Condition`等待队列里，如下，

![Desktop View](/assets/images/20260829/AQS_condition_await_flow.png){: width="600" height="300"}
_await的执行过程_

在`await()`方法中将当前线程挪动到`Condition`等待队列后，还会唤醒`AQS`同步队列中`head`节点的下一个节点。`await()`方法的核心代码如下，

```java
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    Node node = addConditionWaiter();        // step 1
    int savedState = fullyRelease(node);     // step 2
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {           // step 3
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    if (acquireQueued(node, savedState)      // step 4
        && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null)             //step 5
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```

`await()`方法的整体流程如下，
1. 执行`await()`时，会新创建一个节点并放入到`Condition`队列尾部。
2. 然后释放锁，并唤醒`AQS`同步队列中的队首节点的后一个节点。
3. 然后执行`while`循环，将该节点的线程阻塞，直到该节点离开等待队列，重新回到同步队列成为同步节点后，线程才退出`while`循环。
4. 退出循环后，开始调用`acquireQueued()`不断尝试拿锁。
5. 拿到锁后，会清空`Condition`队列中被取消的节点。

创建一个新节点并放入`Condition`队列尾部的工作由`addConditionWaiter()`方法完成，如下，

```java
private Node addConditionWaiter() {
    Node t = lastWaiter;
    // 如果尾节点取消，重新定位尾节点
    if (t != null && t.waitStatus != Node.CONDITION) {
        unlinkCancelledWaiters();
        t = lastWaiter;
    }
    //创建一个新Node，作为等待节点
    Node node = new Node(Thread.currentThread(), Node.CONDITION);
    //将新Node加入等待队列
    if (t == null)
        firstWaiter = node;
    else
        t.nextWaiter = node;
    lastWaiter = node;
    return node;
}
```

##### **signal()唤醒方法**

线程在某个`ConditionObject`对象上调用`signal()`方法后，等待队列中的`firstWaiter`会被加入到同步队列中，等待节点被唤醒，流程如下，

![Desktop View](/assets/images/20260829/AQS_condition_signal_flow.png){: width="600" height="300"}
_signal的执行过程_

`signal()`方法的源码如下，

```java
//唤醒
public final void signal() {
    //如果当前线程不是持有该锁的线程，就抛出异常
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        doSignal(first); //唤醒队首节点
}

//执行唤醒
private void doSignal(Node first) {
    do {
        //出队的代码写得很巧妙，要看仔细
        //first出队，firstWaiter头部指向下一个节点，自己的nextWaiter
        if ((firstWaiter = first.nextWaiter) == null)
            lastWaiter = null; //如果第二节点为空，则尾部也为空
        //将原来头部first的后继置空，help for GC
        first.nextWaiter = null;
    } while (!transferForSignal(first) && (first = firstWaiter) != null);
}

//将被唤醒的节点转移到同步队列
final boolean transferForSignal(Node node) {
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;
    Node p = enq(node); // step 1
    int ws = p.waitStatus;
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread); // step 2: 唤醒线程
    return true;
}
```

`signal()`方法的整体流程如下，·
1. 通过`enq()`方法自旋，将条件队列中的队首节点放入到`AQS`同步队列尾部，并获取它在`AQS`队列中的前驱节点；
2. 如果前驱节点的状态是取消状态，或者设置前驱节点为`Signal`状态失败，就唤醒当前节点的线程；否则节点在同步队列的尾部，参与排队；
3. 同步队列中的线程被唤醒后，表示重新获取了显式锁，然后继续执行`condition.await()`语句后面的临界区代码。

### **ReentrantLock 的抢锁流程**

结合`AbstractQueuedSynchronizer`的模板方法看下`ReentrantLock`的实现过程。

`ReentrantLock`有两种模式，
- 公平锁：按照线程在队列中的排队顺序，先到者先拿到锁；
- 非公平锁：当线程要获取锁时，无视队列顺序直接去抢锁，谁抢到就是谁的。

`ReentrantLock`在同一个时间点只能被一个线程获取，`ReentrantLock`是通过一个`FIFO`的等待队列（`AQS`队列）来管理获取该锁所有线程的。`ReentrantLock`是继承自`Lock`接口实现的独占式可重入锁，并且`ReentrantLock`组合一个`AQS`内部实例完成同步操作。

`ReentrantLock`非公平锁的抢占流程如下，

![Desktop View](/assets/images/20260829/reentrant_lock_acquire_flow_nofair.png){: width="600" height="300"}
_ReentrantLock非公平锁的抢占流程_

`ReentrantLock`为非公平锁实现了一个内部的同步器——`NonfairSync`，其显式锁获取方法`lock()`的源码如下，

```java
static final class NonfairSync extends Sync {
    //非公平锁抢占
    final void lock() {
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
            acquire(1);
    }
    //省略其他
}
```

首先用一个`CAS`操作，判断`state`是否是`0`（表示当前锁未被占用），如果是`0`就把它置为`1`，并且设置当前线程为该锁的独占线程，表示获取锁成功。当多个线程同时尝试占用同一个锁时，`CAS`操作只能保证一个线程操作成功，剩下的只能乖乖去排队。

`ReentrantLock`非公平性即体现在这里：如果占用锁的线程刚释放锁，`state`置为`0`，而排队等待锁的线程还未唤醒，新来的线程就直接抢占了该锁，那么就插队了。

如果非公平抢占没有成功，非公平锁的`lock`会执行模板方法`acquire()`，首先会调用到钩子方法`tryAcquire(arg)`。非公平抢占的钩子方法实现如下，

```java
static final class NonfairSync extends Sync {
    //非公平锁抢占的钩子方法
    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }
    //省略其他
}

abstract static class Sync extends AbstractQueuedSynchronizer {
    final boolean nonfairTryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        //先直接获得锁的状态
        int c = getState();
        if (c == 0) {
            //如果任务队列首节点的线程完了，它会将锁的state设置为0
            //当前抢锁线程的下一步就是直接进行抢占，不管不顾
            //发现state是空的，就直接拿来加锁使用，根本不考虑后面后继者的存在
            if (compareAndSetState(0, acquires)) {
                //1. 利用CAS自旋方式判断当前state确实为0，然后设置成acquire（1）
                //这是原子性的操作，可以保证线程安全
                setExclusiveOwnerThread(current);
                //设置当前执行的线程，直接返回true
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            //2. 当前的线程和执行中的线程是同一个，也就意味着可重入操作
            int nextc = c + acquires;
            if (nextc < 0) // overflow
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            //表示当前锁被1个线程重复获取了nextc次
            return true;
        }
        //否则就是返回false，表示没有尝试成功获取当前锁，进入排队过程
        return false;
    }
    //省略其他
}
```

>非公平同步器`ReentrantLock.NonfairSync`的核心思想就是，当前进程尝试获取锁的时候，如果发现锁的状态位是`0`，就直接尝试将锁拿过来，然后执行`setExclusiveOwnerThread()`，根本不管同步队列中的排队节点。
{: .prompt-tip }

`ReentrantLock`公平锁的抢占流程如下，

![Desktop View](/assets/images/20260829/reentrant_lock_acquire_flow_fair.png){: width="600" height="300"}
_ReentrantLock公平锁的抢占流程_

`ReentrantLock`为公平锁实现了一个内部的同步器——`FairSync`，其显式锁获取方法`lock`的源码如下，

```java
static final class FairSync extends Sync {
    //公平锁抢占的钩子方法
    final void lock() {
        acquire(1);
    }
    //省略其他
}
```

公平同步器`ReentrantLock.FairSync`的核心思想是，通过`AQS`模板方法去进行队列入队操作。

公平锁的`lock`会执行模板方法`acquire`，该方法首先会调用钩子方法`tryAcquire(arg)`。公平抢占的钩子方法实现如下，

```java
static final class FairSync extends Sync {
    //公平抢占的钩子方法
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();                //锁状态
        if (c == 0) {
            if (!hasQueuedPredecessors() &&  //有前驱节点就返回，足够讲义气
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
}
```

公平抢占的钩子方法中，首先判断是否有后继节点，如果有后继节点，并且当前线程不是锁的占有线程，钩子方法就返回`false`，模板方法会进入排队的执行流程，可见公平锁是真正公平的。

`FairSync`进行是否有后继节点的判断，代码如下，

```java
public final boolean hasQueuedPredecessors() {
    Node t = tail;
    Node h = head;
    Node s;
    return h != t &&
        ((s = h.next) == null || s.thread != Thread.currentThread());
}
```

`hasQueuedPredecessors`的执行场景大致如下，
1. `h!=t`不成立，说明`h`队首节点、`t`尾节点要么是同一个节点，要么都是`null`，此时`hasQueuedPredecessors()`返回`false`，表示没有后继节点；
2. `h!=t`成立，进一步检查`head.next`是否为`null`，如果为`null`，就返回`true`。什么情况下`h!=t`同时`h.next==null`呢？有其他线程第一次正在入队时可能会出现。其他线程执行`AQS`的`enq()`方法，`compareAndSetHead(node)`完成，还没执行`tail=head`语句时，此时`t=null`、`head=new Node()`、`head.next=null`；
3. 如果`h!=t`成立，`head.next != null`，判断`head.next`是不是当前线程，如果是就返回`false`，否则返回`true`。

`head`节点是获取到锁的节点，但任意时刻`head`节点可能占用着锁，也可能释放了锁，如果释放了锁，那么此时`state=0`，未被阻塞的`head.next`节点对应的线程在任意时刻都是在自旋地尝试获取锁。
