# Java 基础面试知识卡片

> 适用对象：3年经验 Java 开发者 | 目标：中型互联网公司
> 格式：问答卡片，每个答案适合面试口头回答 2-3 分钟

---

## 一、Java 集合框架

### Q1：HashMap 的底层实现原理是什么？JDK8 前后有什么区别？

**JDK 1.7 及之前：**
- 底层采用 **数组 + 链表** 实现，数组是 `Entry<K,V>[]`。
- 通过 key 的 `hashCode()` 计算哈希值，再经过扰动函数 `(h = key.hashCode()) ^ (h >>> 16)` 得到数组下标。
- 发生哈希冲突时，新元素以**头插法**插入链表。
- 扩容时需要 rehash，多线程下头插法会导致**链表形成环形结构**，造成死循环。

**JDK 1.8：**
- 底层改为 **数组 + 链表 + 红黑树**，节点类型变为 `Node<K,V>`。
- 当链表长度 **>= 8** 且数组长度 **>= 64** 时，链表转化为红黑树，查找时间复杂度从 O(n) 降为 O(log n)。
- 当红黑树节点数 **<= 6** 时，退化回链表。
- 插入改为**尾插法**，解决了扩容时的环形链表问题。
- hash 扰动函数简化为 `(h = key.hashCode()) ^ (h >>> 16)`，减少计算开销。

**关键设计细节：**
- 数组长度必须是 2 的幂次方，这样可以用 `(n - 1) & hash` 代替取模运算，提升性能。
- 默认初始容量 16，负载因子 0.75，阈值 = 容量 x 负载因子。

---

### Q2：HashMap 的扩容机制是怎样的？

**扩容触发条件：**
- 当 `size > threshold`（容量 x 负载因子，默认 16 x 0.75 = 12）时触发扩容。
- 调用 `put()` 方法时检查。

**扩容过程（JDK 1.8）：**
1. 容量翻倍（`newCap = oldCap << 1`），阈值也翻倍。
2. 创建新的 `Node[]` 数组。
3. 遍历旧数组每个桶：
   - 如果只有一个节点，直接计算新位置 `e.hash & (newCap - 1)`。
   - 如果是链表，利用 **`e.hash & oldCap`** 的结果将链表拆成两条：
     - 结果为 0 的留在原索引位。
     - 结果为 1 的移到 `原索引 + oldCap` 的位置。
   - 如果是红黑树，同样拆分，拆分后如果节点数 <= 6 则退化为链表。

**这个设计的巧妙之处：**
- 不需要重新计算 hash，只需要看 hash 值新增的那一位是 0 还是 1，就能确定元素的新位置。
- 避免了 JDK 1.7 中 rehash 的全量重新计算，性能更好。

**面试补充：**
- `HashMap` 的 `resize()` 方法在多线程环境下仍然不安全，可能出现数据覆盖的情况（两个线程同时扩容，一个线程的结果被另一个覆盖）。

---

### Q3：ConcurrentHashMap 的实现原理是什么？JDK8 前后有什么区别？

**JDK 1.7 — 分段锁（Segment）：**
- 底层是 `Segment[]` 数组，每个 Segment 继承自 `ReentrantLock`，本质上是一个小的 HashMap。
- 默认 16 个 Segment，最多支持 16 个线程并发写入（不同 Segment 可以并行）。
- 定位元素需要两次 hash：第一次定位 Segment，第二次定位桶。
- 并发度有限，内存开销较大。

**JDK 1.8 — CAS + synchronized：**
- 抛弃 Segment，采用 **`Node[]` 数组 + 链表 + 红黑树**，和 HashMap 结构类似。
- 使用 **CAS 操作** 进行数组初始化、节点插入（桶为空时）。
- 使用 **`synchronized` 锁住链表/红黑树的头节点**，粒度更细，只锁一个桶而非一个 Segment。
- `size()` 计算使用类似 `LongAdder` 的 `baseCount + CounterCell[]` 方式，避免全局锁。
- `get()` 操作完全无锁，利用 volatile 保证可见性。

**关键对比：**

| 特性 | JDK 1.7 | JDK 1.8 |
|------|---------|---------|
| 锁粒度 | Segment（段锁） | 桶头节点（行锁） |
| 并发度 | 默认 16 | 数组长度 |
| 数据结构 | 数组 + 链表 | 数组 + 链表 + 红黑树 |
| hash 计算 | 两次 | 一次 |
| 锁方式 | ReentrantLock | synchronized + CAS |

---

### Q4：ArrayList 和 LinkedList 的区别及使用场景？

**底层结构：**
- `ArrayList`：基于**动态数组**，内存连续。
- `LinkedList`：基于**双向链表**，内存不连续。

**性能对比：**

| 操作 | ArrayList | LinkedList |
|------|-----------|------------|
| 随机访问 get(i) | O(1) | O(n) |
| 尾部插入 add(e) | 均摊 O(1) | O(1) |
| 中间插入 add(i,e) | O(n)（需移动元素） | O(n)（需遍历定位） |
| 头部插入 addFirst | O(n) | O(1) |
| 删除 remove(i) | O(n) | O(n)（遍历定位） |
| 内存占用 | 较少（连续数组 + 少量预留） | 较多（每个节点有 prev/next 引用） |

**关键细节：**
- `ArrayList` 默认容量 10，扩容为原来的 **1.5 倍**（`oldCapacity + (oldCapacity >> 1)`）。
- `ArrayList` 实现了 `RandomAccess` 接口，`LinkedList` 没有。
- `LinkedList` 同时实现了 `Deque` 接口，可以当双端队列和栈使用。

**使用场景建议：**
- **绝大多数情况选 ArrayList**：缓存友好、随机访问快、内存占用少。
- **选 LinkedList 的场景**：频繁在头部插入/删除、需要双端队列功能（如 LRU 缓存手动实现）。
- 实际开发中，由于 CPU 缓存对连续内存更友好，ArrayList 在大多数场景下性能优于 LinkedList。

---

### Q5：HashMap 为什么线程不安全？

**具体不安全场景：**

1. **JDK 1.7 的扩容死循环：** 多线程同时扩容时，头插法会导致链表节点互相指向，形成环形链表，`get()` 时死循环导致 CPU 100%。

2. **JDK 1.8 的数据覆盖：** 两个线程同时执行 `put()`，计算到同一个桶且桶为空，都执行 CAS 插入，后执行的会覆盖先执行的数据（`++size` 也会丢失计数）。

3. **size 不准确：** `size++` 不是原子操作（读取、加一、写入三步），多线程下会丢失更新。

4. **并发修改异常：** 在迭代过程中，如果另一个线程修改了 HashMap 的结构，会抛出 `ConcurrentModificationException`（fast-fail 机制通过 modCount 检测）。

**面试建议回答方式：**
> HashMap 不安全主要体现在三个方面：JDK7 下扩容可能死循环，JDK8 下并发 put 可能数据覆盖，以及 size 计数不准确。如果需要线程安全，应该使用 ConcurrentHashMap。

---

### Q6：HashSet 的实现原理是什么？

**核心原理：**
- HashSet 内部持有一个 **HashMap** 实例，元素存储在 HashMap 的 **key** 中。
- value 统一使用一个名为 `PRESENT` 的静态常量 `Object` 对象。

**关键源码逻辑：**
```java
// HashSet 内部
private transient HashMap<E,Object> map;
private static final Object PRESENT = new Object();

public boolean add(E e) {
    return map.put(e, PRESENT) == null;  // put返回null说明之前没有这个key
}
```

**去重机制：**
1. 先计算元素的 `hashCode()`，定位到 HashMap 的桶。
2. 如果桶中有元素，再调用 `equals()` 逐个比较。
3. 如果 `equals()` 返回 true，说明元素已存在，不插入。

**注意事项：**
- 存入 HashSet 的对象必须正确重写 `hashCode()` 和 `equals()` 方法。
- HashSet 不保证元素的顺序（底层是 HashMap，顺序由 hash 决定）。
- 如果需要有序，使用 `LinkedHashSet`（内部用 LinkedHashMap，维护插入顺序）或 `TreeSet`（内部用 TreeMap，按自然排序或自定义比较器排序）。

---

### Q7：TreeMap 和 LinkedHashMap 的区别和使用场景？

**TreeMap：**
- 底层是**红黑树**，保证 key 有序（自然排序或 Comparator 自定义排序）。
- 插入、删除、查找的时间复杂度都是 **O(log n)**。
- 实现了 `NavigableMap` 接口，支持 `firstKey()`、`lastKey()`、`subMap()`、`headMap()`、`tailMap()` 等范围操作。
- key 不能为 null（因为需要比较排序）。
- **使用场景：** 需要按 key 排序的场景，如按时间排序的日志、按分数排名的排行榜、区间查询。

**LinkedHashMap：**
- 继承自 HashMap，底层是 **数组 + 链表 + 红黑树**，同时维护一个**双向链表**记录插入顺序。
- 插入、删除、查找的时间复杂度与 HashMap 相同，均摊 O(1)。
- 支持**访问顺序模式**（构造时传 `accessOrder = true`），每次 get/put 会把元素移到链表尾部。
- 可以重写 `removeEldestEntry()` 方法实现 **LRU 缓存**。
- **使用场景：** 需要保持插入顺序的场景、实现 LRU 缓存、按插入顺序遍历。

**LRU 缓存实现示例：**
```java
class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private int capacity;

    public LRUCache(int capacity) {
        super(capacity, 0.75f, true); // accessOrder = true
        this.capacity = capacity;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity; // 超过容量移除最久未访问的元素
    }
}
```

---

## 二、Java 并发编程

### Q1：synchronized 和 ReentrantLock 的区别？

**相同点：**
- 都是可重入锁（同一个线程可以重复获取已持有的锁）。
- 都能保证互斥性和可见性。

**区别：**

| 特性 | synchronized | ReentrantLock |
|------|-------------|---------------|
| 层面 | JVM 层面（关键字） | API 层面（类） |
| 释放锁 | 自动释放（退出代码块/异常） | 必须手动 `unlock()`（通常在 finally 中） |
| 灵活性 | 较低，只能锁代码块 | 较高，支持 `tryLock()`、`lockInterruptibly()`、超时获取 |
| 公平锁 | 不支持（只有非公平锁） | 支持公平锁和非公平锁 |
| 条件变量 | 只能配合 `wait()/notify()`，一个锁一个等待队列 | 支持多个 `Condition`，可以实现精确唤醒 |
| 性能 | JDK 1.6 优化后差距不大 | 略优（高竞争场景） |
| 锁升级 | 无锁 -> 偏向锁 -> 轻量级锁 -> 重量级锁 | 无锁升级概念 |

**synchronized 的锁升级（JDK 1.6）：**
1. **无锁**：没有线程竞争。
2. **偏向锁**：第一个线程 CAS 将线程 ID 写入对象头 Mark Word，后续该线程进入无需 CAS。
3. **轻量级锁**：出现竞争时，偏向锁升级。线程在栈帧中创建 Lock Record，CAS 替换 Mark Word。获取失败则自旋。
4. **重量级锁**：自旋失败，升级为重量级锁，依赖操作系统 Mutex Lock，线程阻塞/唤醒需要用户态和内核态切换。

**选择建议：**
- 简单场景用 `synchronized`（代码简洁、JVM 自动优化）。
- 需要 `tryLock`、公平锁、多条件变量、响应中断等高级功能时用 `ReentrantLock`。

---

### Q2：volatile 关键字的作用和原理是什么？

**两大作用：**

1. **保证可见性：**
   - 当一个线程修改了 volatile 变量，新值会立即刷新到主内存。
   - 其他线程读取该变量时，会直接从主内存读取最新值，而不是使用工作内存中的缓存。
   - 底层通过**内存屏障**（Memory Barrier）实现：写操作前插入 StoreStore 屏障，写操作后插入 StoreLoad 屏障；读操作前插入 LoadLoad 屏障，读操作后插入 LoadStore 屏障。

2. **禁止指令重排序：**
   - 编译器和 CPU 可能对指令进行重排序优化，volatile 通过内存屏障阻止特定类型的重排序。
   - **经典案例 — 双重检查锁单例：**
     ```java
     private static volatile Singleton instance; // 必须用 volatile
     public static Singleton getInstance() {
         if (instance == null) {                 // 第一次检查
             synchronized (Singleton.class) {
                 if (instance == null) {          // 第二次检查
                     instance = new Singleton();  // 非原子：1.分配内存 2.初始化 3.引用赋值
                 }                                // 没有volatile可能重排为1->3->2
             }                                    // 其他线程可能拿到未初始化的对象
         }
         return instance;
     }
     ```

**volatile 不保证原子性：**
- `volatile int count; count++` 仍然不安全，因为 `count++` 是三步操作（读、加、写）。
- 需要原子性应使用 `AtomicInteger` 或加锁。

**适用场景：**
- 状态标志位（如 `volatile boolean running = true`）。
- 双重检查锁定。
- 一写多读的场景。

---

### Q3：CAS 原理及 ABA 问题是什么？

**CAS 原理：**
- CAS（Compare And Swap）是一种**乐观锁**机制，包含三个操作数：内存地址 V、期望值 A、新值 B。
- 当且仅当 V 的当前值等于 A 时，才将 V 更新为 B，否则不操作。这是一条 CPU 原子指令（如 x86 的 `cmpxchg`）。
- Java 中通过 `Unsafe` 类提供 CAS 操作，`AtomicInteger`、`AtomicReference` 等原子类底层都基于 CAS。

**CAS 的优点：**
- 无锁操作，避免了线程切换的开销。
- 低竞争场景下性能优于 synchronized。

**CAS 的三大问题：**

1. **ABA 问题：**
   - 线程 1 读取值为 A，线程 2 将值改为 B 再改回 A，线程 1 CAS 成功，但中间发生了变化。
   - **解决方案：** 使用 `AtomicStampedReference`（带版本号）或 `AtomicMarkableReference`（带标记位）。每次修改时版本号递增，即使值相同，版本号不同也会 CAS 失败。

2. **自旋开销：**
   - CAS 失败后通常会自旋重试，如果竞争激烈，长时间自旋会浪费 CPU 资源。

3. **只能保证单个变量的原子性：**
   - 多个变量需要用 `AtomicReference` 封装成一个对象，或者使用锁。

---

### Q4：线程池的核心参数和工作流程是什么？

**7 个核心参数（ThreadPoolExecutor 构造方法）：**

| 参数 | 类型 | 说明 |
|------|------|------|
| corePoolSize | int | 核心线程数（即使空闲也不回收，除非设置 allowCoreThreadTimeOut） |
| maximumPoolSize | int | 最大线程数 |
| keepAliveTime | long | 非核心线程空闲存活时间 |
| unit | TimeUnit | 存活时间的单位 |
| workQueue | BlockingQueue | 任务等待队列 |
| threadFactory | ThreadFactory | 线程工厂（可自定义线程名称，便于排查问题） |
| handler | RejectedExecutionHandler | 拒绝策略 |

**工作流程（重要，面试必问）：**

```
提交任务
  |
  v
当前线程数 < corePoolSize？ ——是——> 创建核心线程执行任务
  |
  否
  |
  v
workQueue 未满？ ——是——> 任务入队等待
  |
  否（队列满了）
  |
  v
当前线程数 < maximumPoolSize？ ——是——> 创建非核心线程执行任务
  |
  否（已达最大线程数）
  |
  v
执行拒绝策略（handler）
```

**常见队列类型：**
- `LinkedBlockingQueue`：无界/有界链表队列（默认无界，**注意 OOM 风险**）。
- `ArrayBlockingQueue`：有界数组队列（推荐，可控资源）。
- `SynchronousQueue`：不存储元素，每个 put 必须等待一个 take（CachedThreadPool 使用）。
- `PriorityBlockingQueue`：优先级队列。

**线程池配置建议：**
- **CPU 密集型**：`N + 1`（N 为 CPU 核心数），减少线程切换。
- **IO 密集型**：`2N` 或 `N / (1 - 阻塞系数)`，因为线程大部分时间在等待 IO。

---

### Q5：线程池的拒绝策略有哪些？

当线程池和队列都满了，会触发拒绝策略。JDK 提供 4 种内置策略：

| 策略 | 类名 | 行为 |
|------|------|------|
| 中止（默认） | `AbortPolicy` | 抛出 `RejectedExecutionException` 异常 |
| 调用者运行 | `CallerRunsPolicy` | 由提交任务的线程（调用者线程）自己执行该任务，起到**反压**效果 |
| 丢弃最老 | `DiscardOldestPolicy` | 丢弃队列头部最老的任务，将当前任务重新入队 |
| 丢弃 | `DiscardPolicy` | 静默丢弃任务，不抛异常 |

**实际生产中的做法：**
- 一般不使用默认策略，而是**自定义拒绝策略**，记录日志、持久化任务或上报监控：
```java
new ThreadPoolExecutor.CallerRunsPolicy() // 简单反压
// 或自定义
(r, executor) -> {
    log.warn("线程池已满，任务被拒绝: {}", r);
    // 持久化到数据库或MQ，后续补偿执行
}
```

**面试加分点 — CallerRunsPolicy 的巧妙之处：**
- 调用者线程去执行任务，调用者被阻塞无法提交新任务，自然形成**限流/反压**，不会丢失任务。

---

### Q6：CountDownLatch、CyclicBarrier、Semaphore 的使用场景？

**CountDownLatch（倒计数器）：**
- 一个或多个线程等待其他 N 个线程完成。
- 构造函数传入 count，每个完成的线程调用 `countDown()`，等待的线程调用 `await()` 阻塞直到 count 变为 0。
- **一次性使用**，计数到 0 不可重置。
- **场景：** 主线程等待多个子任务完成后汇总结果；多个服务启动完成后才开始处理请求。

```java
CountDownLatch latch = new CountDownLatch(3);
// 3个子线程各自完成任务后 latch.countDown();
latch.await(); // 主线程等待
// 汇总结果
```

**CyclicBarrier（循环栅栏）：**
- N 个线程互相等待，全部到达屏障点后一起继续执行。
- 所有线程调用 `await()` 阻塞，当到达的线程数等于 parties 时，所有线程同时放行。
- **可重复使用**（`reset()` 或自动重置），支持 barrier action。
- **场景：** 多线程计算数据分段，全部分段计算完成后合并；模拟并发测试（所有线程同时起跑）。

**Semaphore（信号量）：**
- 控制同时访问某个资源的线程数量（限流）。
- `acquire()` 获取许可，`release()` 释放许可。
- **场景：** 数据库连接池限流、限流接口并发数、停车场车位管理。

```java
Semaphore semaphore = new Semaphore(3); // 最多3个线程同时访问
semaphore.acquire();
try {
    // 访问资源
} finally {
    semaphore.release();
}
```

**三者对比：**

| 特性 | CountDownLatch | CyclicBarrier | Semaphore |
|------|---------------|---------------|-----------|
| 用途 | 等待多个线程完成 | 多个线程互相等待 | 控制并发访问数 |
| 可重用 | 否 | 是 | 是 |
| 关系 | 1对N | N对N | N对资源 |

---

### Q7：ThreadLocal 原理及内存泄漏问题？

**原理：**
- 每个 `Thread` 对象内部持有一个 `ThreadLocalMap`（`Thread.threadLocals` 字段）。
- `ThreadLocalMap` 的 key 是 `ThreadLocal` 实例的**弱引用**，value 是实际存储的值。
- 调用 `threadLocal.set(value)` 时，以当前 ThreadLocal 对象为 key，将 value 存入当前线程的 ThreadLocalMap 中。
- 调用 `threadLocal.get()` 时，从当前线程的 ThreadLocalMap 中以当前 ThreadLocal 为 key 取值。

**内存泄漏问题：**
- `ThreadLocalMap` 的 key 是弱引用，当 ThreadLocal 对象没有强引用时，GC 后 key 变为 null。
- 但 value 是**强引用**，如果线程长期存活（如线程池中的线程），value 永远不会被回收。
- 形成 `null -> value` 的无效条目，造成内存泄漏。

**ThreadLocalMap 的防护措施：**
- `set()`、`get()`、`remove()` 方法内部会清理部分 key 为 null 的条目（启发式清理）。
- 但这不能保证完全清理。

**最佳实践：**
- **使用完 ThreadLocal 后务必在 finally 中调用 `remove()`**：
```java
private static final ThreadLocal<UserContext> contextHolder = new ThreadLocal<>();
try {
    contextHolder.set(userContext);
    // 业务逻辑
} finally {
    contextHolder.remove(); // 必须清理，尤其在线程池环境下
}
```

**常见使用场景：**
- 存储用户登录信息（如 Spring 的 `RequestContextHolder`）。
- 数据库连接/事务管理（每个线程独立连接）。
- `SimpleDateFormat` 线程不安全，用 ThreadLocal 让每个线程拥有独立实例。

---

### Q8：AQS 的原理是什么？

**AQS（AbstractQueuedSynchronizer）是 Java 并发包的基础框架。**

**核心结构：**
- 一个 `volatile int state` 变量，表示同步状态（如锁的持有计数、信号量许可数）。
- 一个 **FIFO 双向等待队列**（CLH 变体），用于管理等待线程。
- 通过 CAS 操作修改 state，通过 `LockSupport.park()/unpark()` 阻塞/唤醒线程。

**两种模式：**
- **独占模式（Exclusive）：** 同一时刻只有一个线程能获取锁，如 `ReentrantLock`。
- **共享模式（Shared）：** 多个线程可以同时获取，如 `Semaphore`、`CountDownLatch`、`ReadWriteLock` 的读锁。

**ReentrantLock 基于 AQS 的实现：**
- state = 0 表示未锁定，state > 0 表示重入次数。
- 非公平锁：新线程直接 CAS 尝试获取，失败才入队。
- 公平锁：新线程先检查队列中是否有等待者，有则排队。
- 获取失败时，将线程封装为 Node 加入等待队列尾部，然后 `park()` 阻塞。
- 释放锁时，`state` 减到 0，`unpark()` 队列中下一个线程。

**基于 AQS 的常见同步器：**
- `ReentrantLock`：独占模式，state 表示重入次数。
- `Semaphore`：共享模式，state 表示剩余许可数。
- `CountDownLatch`：共享模式，state 表示计数值。
- `ReentrantReadWriteLock`：state 高 16 位表示读锁持有数，低 16 位表示写锁重入数。

---

### Q9：死锁的产生条件和排查方法？

**四个必要条件（缺一不可）：**
1. **互斥条件：** 资源同时只能被一个线程持有。
2. **持有并等待：** 线程持有至少一个资源，同时请求另一个被其他线程持有的资源。
3. **不可抢占：** 资源只能由持有者主动释放，不能被强制夺取。
4. **循环等待：** 存在一条线程等待链，形成环形（T1 等 T2，T2 等 T3，...，Tn 等 T1）。

**经典死锁代码：**
```java
// 线程1：先锁A，再锁B
synchronized (lockA) {
    Thread.sleep(100);
    synchronized (lockB) { ... }
}
// 线程2：先锁B，再锁A
synchronized (lockB) {
    Thread.sleep(100);
    synchronized (lockA) { ... }
}
```

**排查方法：**

1. **jstack（最常用）：**
   - `jps` 找到 Java 进程 PID。
   - `jstack <pid>` 导出线程堆栈，会自动检测并打印 `Found one Java-level deadlock` 信息。

2. **jconsole / jvisualvm：**
   - 图形化工具，内置死锁检测功能，点击"检测死锁"即可查看。

3. **arthas（线上推荐）：**
   - `thread -b` 直接定位阻塞其他线程的线程。

**预防死锁的方法：**
- 破坏循环等待：所有线程按**相同顺序**获取锁。
- 破坏持有并等待：使用 `tryLock()` 超时机制，获取失败则释放已持有的锁。
- 使用 `Lock.lockInterruptibly()` 支持响应中断。

---

### Q10：Java 内存模型（JMM）是什么？

**JMM 是 Java 定义的并发编程内存访问规范（JSR-133），不是 JVM 的内存区域划分。**

**核心概念：**
- 每个线程有自己的**工作内存**（对应 CPU 缓存），存储该线程使用的变量副本。
- 所有线程共享**主内存**（对应堆内存），存储所有变量。
- 线程对变量的操作必须在工作内存中进行，不能直接操作主内存。

**三大特性：**

1. **原子性：** 一个操作要么全部完成，要么全部不做。
   - 基本数据类型的读写是原子的。
   - `long/double` 的写操作在 32 位 JVM 上可能不是原子的（64 位 JVM 已解决）。
   - 保证原子性：`synchronized`、`Lock`、`Atomic` 类。

2. **可见性：** 一个线程对变量的修改，其他线程能立即看到。
   - 保证可见性：`volatile`、`synchronized`（解锁前刷新到主内存）、`final`。

3. **有序性：** 程序执行的顺序与代码顺序一致。
   - 编译器和 CPU 可能重排序指令以优化性能。
   - 保证有序性：`volatile`（内存屏障）、`happens-before` 规则。

**happens-before 原则（重点）：**
- **程序顺序规则：** 同一个线程中，前面的操作 happens-before 后面的操作。
- **监视器锁规则：** unlock 操作 happens-before 后续对同一个锁的 lock 操作。
- **volatile 规则：** volatile 写 happens-before 后续的 volatile 读。
- **线程启动规则：** `Thread.start()` happens-before 该线程中的任何操作。
- **线程终止规则：** 线程中的所有操作 happens-before `Thread.join()` 返回。
- **传递性：** A happens-before B，B happens-before C，则 A happens-before C。

---

## 三、JVM

### Q1：JVM 内存区域是如何划分的？

**运行时数据区（Runtime Data Areas）：**

```
┌──────────────────────────────────────────────────┐
│                    JVM 内存                       │
├─────────────────────┬────────────────────────────┤
│    线程私有区域       │       线程共享区域          │
├─────────────────────┼────────────────────────────┤
│ 程序计数器(PC)       │ 堆（Heap）                 │
│ 虚拟机栈(VM Stack)   │   - 新生代(Eden+S0+S1)     │
│ 本地方法栈           │   - 老年代(Old Gen)         │
│                     │ 方法区(Metaspace/PermGen)   │
└─────────────────────┴────────────────────────────┘
```

**各区域详解：**

1. **程序计数器（PC Register）：**
   - 记录当前线程正在执行的字节码指令地址。
   - 线程私有，唯一一个**不会发生 OOM** 的区域。
   - 执行 native 方法时，PC 值为 undefined。

2. **虚拟机栈（VM Stack）：**
   - 线程私有，每个方法调用创建一个**栈帧**（Stack Frame）。
   - 栈帧包含：局部变量表、操作数栈、动态链接、方法返回地址。
   - 递归过深或栈帧过多会抛出 `StackOverflowError`；动态扩展失败会抛出 `OOM`。

3. **本地方法栈（Native Method Stack）：**
   - 为 native 方法服务，和虚拟机栈类似。

4. **堆（Heap）：**
   - 线程共享，存放对象实例和数组，是 **GC 的主要区域**。
   - 分为新生代（Eden 80% + Survivor0 10% + Survivor1 10%）和老年代。
   - 对象优先在 Eden 区分配，经过多次 Minor GC 仍然存活则晋升到老年代。

5. **方法区（Method Area / Metaspace）：**
   - JDK 1.7 及之前叫**永久代**（PermGen），受 `-XX:MaxPermSize` 限制。
   - JDK 1.8 改为**元空间**（Metaspace），使用**本地内存**（Native Memory），不再受 JVM 堆大小限制。
   - 存储：类信息、常量池、静态变量、JIT 编译后的代码。

6. **直接内存（Direct Memory）：**
   - 不属于 JVM 运行时数据区，是 NIO 通过 `ByteBuffer.allocateDirect()` 分配的堆外内存。
   - 受 `-XX:MaxDirectMemorySize` 限制，也可能出现 OOM。

---

### Q2：垃圾回收算法有哪些？

1. **标记-清除（Mark-Sweep）：**
   - 先标记所有存活对象，再清除未标记对象。
   - **缺点：** 产生内存碎片，不连续的空间可能无法分配大对象。

2. **标记-复制（Mark-Copy）：**
   - 将内存分为两块，每次只用一块，GC 时将存活对象复制到另一块。
   - **新生代采用此算法**（Eden + 两个 Survivor，实际使用率 90%）。
   - **优点：** 无碎片、分配快（指针碰撞）。
   - **缺点：** 浪费一半内存（新生代通过 8:1:1 比例优化为只浪费 10%）。

3. **标记-整理（Mark-Compact）：**
   - 先标记存活对象，然后将存活对象向一端移动，清理边界外的空间。
   - **老年代常用**，避免碎片。
   - **缺点：** 移动对象需要更新引用，STW 时间较长。

4. **分代收集（Generational Collection）：**
   - 根据对象存活周期选择不同算法，是现代 JVM 的核心策略。
   - 新生代：对象存活率低，用**复制算法**（Minor GC）。
   - 老年代：对象存活率高，用**标记-清除**或**标记-整理**（Major GC / Full GC）。

**面试加分点 — 对象晋升老年代的条件：**
- 年龄达到阈值（默认 15，`-XX:MaxTenuringThreshold`）。
- 大对象直接进入老年代（`-XX:PretenureSizeThreshold`）。
- Survivor 区放不下的对象直接晋升。
- 动态年龄判定：Survivor 中相同年龄的对象总大小超过 Survivor 空间的一半，则 >= 该年龄的对象直接晋升。

---

### Q3：常见的垃圾收集器有哪些？（CMS、G1、ZGC）

**Serial / Serial Old（单线程）：**
- 单线程收集器，GC 时 STW（Stop The World）。
- 适合 Client 模式、小型应用。

**Parallel Scavenge / Parallel Old（吞吐量优先）：**
- 多线程，关注**吞吐量**（运行时间 / (运行时间 + GC 时间)）。
- JDK 1.8 默认。

**CMS（Concurrent Mark Sweep）— 老年代，低延迟：**

四个阶段：
1. **初始标记（STW）：** 标记 GC Roots 直接关联的对象，很快。
2. **并发标记：** 从 GC Roots 出发遍历整个对象图，与用户线程并发。
3. **重新标记（STW）：** 修正并发标记阶段因用户线程运行而变化的标记。
4. **并发清除：** 清除未标记对象，与用户线程并发。

**CMS 缺点：**
- 对 CPU 敏感（并发阶段占用 CPU）。
- 无法处理**浮动垃圾**（并发清除阶段新产生的垃圾）。
- 基于标记-清除，会产生**内存碎片**，可能触发 Full GC（退化为 Serial Old）。

**G1（Garbage First）— JDK 9 默认：**

- 将堆划分为多个大小相等的 **Region**（默认约 2048 个），每个 Region 可以是 Eden、Survivor、Old 或 Humongous（存放大对象）。
- **混合回收（Mixed GC）：** 一次 GC 可以回收部分新生代和部分老年代 Region。
- **核心流程：**
  1. 初始标记（STW，借助 Minor GC 一起完成）。
  2. 并发标记。
  3. 最终标记（STW，处理 SATB 记录）。
  4. 筛选回收（STW，按回收价值和用户设定的停顿时间目标 `-XX:MaxGCPauseMillis` 选择 Region 回收）。
- 使用**记忆集（Remembered Set）** 记录跨 Region 引用，避免全堆扫描。
- 整体基于**标记-整理**，局部基于**复制算法**，不产生碎片。

**ZGC（JDK 15 正式发布，超低延迟）：**
- 目标：**停顿时间不超过 1ms**，与堆大小无关（支持 TB 级堆）。
- **核心技术：**
  - **染色指针（Colored Pointers）：** 在指针中嵌入标记信息（Marked0、Marked1、Remapped、Finalizable），不需要额外内存。
  - **读屏障（Load Barrier）：** 读取对象引用时检查染色指针，发现过期引用则自愈。
  - **并发转移：** 对象转移阶段几乎完全并发。
- JDK 16 引入**分代 ZGC**（JEP 473 在 JDK 23 中进一步完善），性能更优。

**收集器选择建议：**
- 小型应用 / 后台服务：Parallel Scavenge + Parallel Old。
- 互联网应用（注重响应时间）：G1（JDK 9+ 默认）。
- 超大堆 + 超低延迟要求：ZGC（JDK 15+）。

---

### Q4：如何判断对象可以被回收？

**两种判断方法：**

**1. 引用计数法（Java 不使用）：**
- 每个对象有一个引用计数器，被引用 +1，引用失效 -1，为 0 则回收。
- **致命缺点：** 无法解决**循环引用**问题（A 引用 B，B 引用 A，计数都不为 0）。
- Python 使用此方法（配合其他机制解决循环引用）。

**2. 可达性分析（Java 使用）：**
- 从一组 **GC Roots** 开始向下搜索引用链，不可达的对象就是可回收的。
- **GC Roots 包括：**
  - 虚拟机栈中引用的对象（局部变量）。
  - 方法区中静态变量引用的对象。
  - 方法区中常量引用的对象。
  - 本地方法栈中 JNI（Native 方法）引用的对象。
  - 被 synchronized 持有的对象。

**四种引用类型：**

| 引用类型 | 回收时机 | 用途 |
|---------|---------|------|
| 强引用 | 永不回收（只要可达） | 普通对象引用 `Object obj = new Object()` |
| 软引用 SoftReference | 内存不足时回收 | 缓存（内存够就不回收，不够就回收） |
| 弱引用 WeakReference | 下次 GC 时回收 | ThreadLocalMap 的 key、WeakHashMap |
| 虚引用 PhantomReference | 随时回收，无法通过它获取对象 | 跟踪对象被回收的活动，配合 ReferenceQueue |

**finalize() 方法（不推荐）：**
- 不可达对象在被回收前，如果重写了 `finalize()` 且未被调用过，会被放入 F-Queue 队列。
- Finalizer 线程执行 `finalize()`，如果在此方法中重新建立引用，对象可以"复活"。
- **不推荐原因：** 执行时间不确定、性能差、可能导致对象复活引发更多问题。JDK 9 已标记为 Deprecated。

---

### Q5：类加载机制（双亲委派模型）是什么？

**类加载过程（5 个阶段）：**

1. **加载（Loading）：** 通过全限定名获取二进制字节流，转化为方法区的运行时数据结构，生成 Class 对象。
2. **验证（Verification）：** 验证字节码格式、语义、字节码指令、符号引用的正确性。
3. **准备（Preparation）：** 为类的**静态变量**分配内存并设置**零值**（`static final` 常量直接赋值）。
4. **解析（Resolution）：** 将常量池中的符号引用替换为直接引用。
5. **初始化（Initialization）：** 执行类构造器 `<clinit>()`，即静态变量赋值和静态代码块。

**双亲委派模型：**

```
Bootstrap ClassLoader（启动类加载器，rt.jar）
        ↑
Extension ClassLoader（扩展类加载器，ext/*.jar）
        ↑
Application ClassLoader（应用类加载器，classpath）
        ↑
Custom ClassLoader（自定义类加载器）
```

**工作流程：**
- 收到类加载请求时，**先委托父加载器处理**，父加载器无法加载时，子加载器才自己尝试加载。
- 最终都会传递到 Bootstrap ClassLoader。

**为什么需要双亲委派？**
- **安全性：** 防止核心类被篡改。自定义一个 `java.lang.String` 类不会被加载，因为 Bootstrap 会优先加载 rt.jar 中的 String。
- **避免重复加载：** 父加载器已加载的类，子加载器不会再加载。

**打破双亲委派的场景：**
- **JNDI / SPI 机制：** 使用**线程上下文类加载器**（Thread Context ClassLoader），让父加载器反向委托子加载器加载实现类（如 JDBC Driver）。
- **OSGi：** 模块化框架，使用网状委派模型。
- **Tomcat：** 每个 Web 应用有独立的 WebAppClassLoader，优先加载自己目录下的类（子优先策略），实现应用隔离。
- **热部署/热加载：** 自定义 ClassLoader 每次重新加载类。

**自定义 ClassLoader 示例：**
```java
public class MyClassLoader extends ClassLoader {
    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        byte[] data = loadClassData(name); // 从文件/网络读取字节码
        return defineClass(name, data, 0, data.length);
    }
}
```

---

### Q6：JVM 调优参数有哪些？

**堆和内存相关：**

| 参数 | 说明 |
|------|------|
| `-Xms2g` | 初始堆大小（建议和 -Xmx 一致，避免动态扩缩） |
| `-Xmx2g` | 最大堆大小 |
| `-Xmn1g` | 新生代大小 |
| `-Xss256k` | 每个线程的栈大小 |
| `-XX:MetaspaceSize=256m` | 元空间初始大小 |
| `-XX:MaxMetaspaceSize=512m` | 元空间最大限制 |
| `-XX:NewRatio=2` | 老年代:新生代 = 2:1 |
| `-XX:SurvivorRatio=8` | Eden:Survivor = 8:1:1 |
| `-XX:MaxDirectMemorySize=512m` | 直接内存最大值 |

**GC 收集器选择：**

| 参数 | 说明 |
|------|------|
| `-XX:+UseG1GC` | 使用 G1 收集器 |
| `-XX:+UseConcMarkSweepGC` | 使用 CMS（JDK 14 已移除） |
| `-XX:+UseZGC` | 使用 ZGC（JDK 15+） |
| `-XX:MaxGCPauseMillis=200` | G1 目标最大停顿时间 |
| `-XX:ConcGCThreads=4` | 并发 GC 线程数 |

**GC 日志相关：**

| 参数 | 说明 |
|------|------|
| `-XX:+PrintGCDetails` | 打印 GC 详情（JDK 8） |
| `-Xloggc:/path/gc.log` | GC 日志输出路径（JDK 8） |
| `-Xlog:gc*:file=gc.log:time` | JDK 9+ 统一日志框架 |

**OOM 相关：**

| 参数 | 说明 |
|------|------|
| `-XX:+HeapDumpOnOutOfMemoryError` | OOM 时自动 dump 堆 |
| `-XX:HeapDumpPath=/path/dump.hprof` | dump 文件路径 |
| `-XX:ErrorFile=/path/hs_err.log` | 致命错误日志路径 |

**调优经验值参考（中型互联网应用，4核8G机器）：**
```bash
-Xms4g -Xmx4g              # 堆固定4G，避免扩缩抖动
-Xmn2g                     # 新生代2G
-XX:MetaspaceSize=256m
-XX:MaxMetaspaceSize=512m
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/data/logs/dump.hprof
-Xlog:gc*:file=/data/logs/gc.log:time
```

---

### Q7：线上 OOM 问题如何排查？

**OOM 常见类型：**

| 错误信息 | 原因 | 常见场景 |
|---------|------|---------|
| `Java heap space` | 堆内存不足 | 大对象、内存泄漏、并发量大 |
| `GC overhead limit exceeded` | GC 耗时超过 98% 但回收不到 2% 的内存 | 内存泄漏晚期 |
| `Metaspace` | 元空间不足 | 动态生成大量类（CGLIB、反射） |
| `Direct buffer memory` | 直接内存溢出 | NIO 大量使用 DirectByteBuffer |
| `Unable to create new native thread` | 无法创建新线程 | 线程泄漏、线程池配置过大 |

**排查步骤（实战流程）：**

**第一步：保留现场**
- 提前配置 `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/data/dump.hprof`。
- 如果没有提前配置，用 `jmap -dump:format=b,file=dump.hprof <pid>` 手动导出。

**第二步：分析堆转储文件**
- 工具：MAT（Memory Analyzer Tool）、VisualVM、JProfiler。
- 查看 **Dominator Tree**（支配树），找到占用内存最大的对象。
- 查看 **Leak Suspects**（泄漏嫌疑人报告），MAT 会自动分析可疑泄漏点。
- 对比多次 dump 文件，看哪些对象持续增长。

**第三步：定位代码**
- 从泄漏对象的 GC Roots 引用链追溯到具体代码位置。
- 常见原因：
  - 集合持续增长未清理（如 static Map 做缓存无上限）。
  - ThreadLocal 未 remove。
  - 数据库查询未加分页/限制，查出全量数据。
  - 大文件读取未使用流式处理。

**第四步：修复验证**
- 修复代码后，压测验证，观察 GC 日志和内存曲线是否恢复正常（锯齿形）。

**常用命令速查：**
```bash
jmap -heap <pid>              # 查看堆配置和使用情况
jmap -histo:live <pid>        # 查看存活对象统计（会触发Full GC）
jstat -gcutil <pid> 1000      # 每秒输出一次GC摘要
```

---

### Q8：线上 CPU 飙高如何排查？

**排查步骤（实战流程）：**

**第一步：找到占用 CPU 最高的 Java 进程**
```bash
top                           # 找到 CPU 最高的进程 PID
# 或
top -c                        # 显示完整命令行，确认是 Java 进程
```

**第二步：找到进程中 CPU 最高的线程**
```bash
top -Hp <pid>                 # 列出该进程下所有线程，找到 CPU 最高的线程 TID
```

**第三步：转换线程 ID 为十六进制**
```bash
printf "%x\n" <tid>           # 将十进制 TID 转为十六进制 nid
```

**第四步：导出线程堆栈并定位**
```bash
jstack <pid> | grep -A 30 <nid>   # 用十六进制线程ID在堆栈中搜索
```

**第五步：分析堆栈信息**

**常见原因和对应的堆栈特征：**

1. **死循环：**
   - 堆栈反复指向同一段业务代码（如 `while(true)` 中 HashMap 扩容导致的环形链表）。

2. **频繁 GC：**
   - 堆栈中出现 `GC task thread`、`VM Thread`。
   - 用 `jstat -gcutil <pid> 1000` 确认是否 Full GC 频繁（FGC 计数快速增长）。
   - 原因通常是内存不足、对象创建过快，需要结合 OOM 排查思路分析。

3. **正则回溯（ReDoS）：**
   - 堆栈中出现 `java.util.regex.Pattern$xxx.match`。
   - 复杂的正则表达式在特定输入下产生指数级回溯。

4. **锁竞争：**
   - 大量线程处于 `BLOCKED` 或 `WAITING` 状态。
   - 堆栈中出现 `waiting to lock` 或 `locked`。

5. **线程泄漏：**
   - 线程数持续增长（数千甚至数万），大量线程处于 RUNNABLE 或 TIMED_WAITING。
   - `jstack <pid> | grep -c "java.lang.Thread.State"` 统计线程数。

**工具推荐：**
- **arthas（阿里开源，线上排查首选）：**
  ```bash
  thread -n 3           # 查看 CPU 最高的 3 个线程及堆栈
  thread -b             # 查找阻塞其他线程的线程（死锁检测）
  dashboard             # 实时查看线程、内存、GC 概况
  profiler start/stop   # 火焰图分析
  ```

**预防措施：**
- 配置完善的监控告警（Prometheus + Grafana / 阿里云 ARMS）。
- 压测时关注 CPU、GC、线程数等指标。
- 代码 Review 关注：死循环风险、正则复杂度、线程池配置合理性。

---

> **复习建议：** 每个主题至少完整过 2-3 遍，重点理解原理而非死记硬背。面试时结合项目经验举例说明，效果更佳。
