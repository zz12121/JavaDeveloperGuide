
# JMM对并发编程的影响

## 核心结论

Java 内存模型（JMM）定义了多线程环境下变量的**可见性、有序性和原子性**规则。理解 JMM 是正确编写并发程序的基础——大多数并发 Bug（读到过期数据、指令重排序、操作丢失）都可以通过 JMM 原理解释。

## 深度解析

### JMM 如何影响日常编程

| 并发问题 | JMM 解释 | 解决方案 |
|---------|---------|---------|
| 读到过期数据 | 线程缓存未刷新主存 | volatile / synchronized |
| 指令重排序导致错误 | 编译器/CPU 重排序 | volatile（内存屏障）/ happens-before |
| i++ 结果不对 | 非原子操作（读-改-写） | synchronized / AtomicXxx / Lock |
| 单例 DCL 不安全 | 对象创建可能被重排序 | volatile 修饰实例 |
| 线程停止不下来 | stop 标志不可见 | volatile / interrupt |

### happens-before 规则（实用版）

```
程序顺序规则：  同一线程中前面的操作先于后面的操作
volatile 规则： volatile 写先于后续的 volatile 读
锁规则：       unlock 先于后续的同一锁的 lock
线程启动规则：  Thread.start() 先于线程内所有操作
线程终止规则：  线程所有操作先于 Thread.join() 返回
传递性：       A 先于 B，B 先于 C → A 先于 C
```

### 实际影响：DCL 单例

```java
public class Singleton {
    // 没有 volatile 时，可能发生：
    // 线程A：分配内存 → 赋值引用（非null但未初始化）→ 线程B判断非null直接返回 → 使用未初始化对象
    private static volatile Singleton instance; // volatile 禁止重排序

    public static Singleton getInstance() {
        if (instance == null) {               // 第一次检查（无锁）
            synchronized (Singleton.class) {   // 加锁
                if (instance == null) {        // 第二次检查
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

### 实际影响：状态标志

```java
// ❌ 不用 volatile
private boolean running = true;
// 其他线程可能永远看不到 running = false

// ✅ 用 volatile
private volatile boolean running = true;
// volatile 写后强制刷新主存，读前 invalidate 缓存

// ✅ 用 AtomicBoolean
private AtomicBoolean running = new AtomicBoolean(true);
```

### 实际影响：并发容器选择

```java
// HashMap 非线程安全 → JMM 下 put/get 可能互相覆盖
// Hashtable 全表锁 → JMM 保证安全但性能差
// ConcurrentHashMap → JMM + CAS + synchronized 保证安全且高性能
```

### 完整 happens-before 规则（8条）

```
1. 程序顺序规则：  同一线程中，前面的操作 hb 后面的操作
2. volatile 规则： volatile 变量的写操作 hb 后续的读操作
3. 锁规则：       unlock 操作 hb 后续对同一锁的 lock 操作
4. 线程启动规则：  Thread.start() hb 线程内所有操作
5. 线程终止规则：  线程所有操作 hb Thread.join() 返回
6. 线程中断规则：  Thread.interrupt() hb 被中断线程检测到中断
7. 对象终结规则：  对象构造函数结束 hb finalize() 开始
8. 传递性：       A hb B，B hb C → A hb C
```

**实用判断法**：
- 两个操作之间能找到一条 happens-before 链 → 线程安全（结果可见）
- 找不到 → 存在竞态条件，需要额外同步

### volatile 内存屏障

`volatile` 不止保证可见性，还通过**内存屏障**禁止重排序：

```
volatile 写（StoreStore + StoreLoad）：
  写之前不能重排序（StoreStore：前面所有写完才写 volatile）
  写之后的读不能重排序（StoreLoad：写完后读必须从主存）

volatile 读（LoadLoad + LoadStore）：
  读之后的操作不能重排序（LoadLoad：读完才能读后续）
  读之后的写不能重排序（LoadStore：读完才能写后续）
```

```java
// 实际场景：用 volatile 做发布屏障
class UnsafeLazyInit {
    private static Resource resource;  // 没有 volatile
    
    // 问题：resource = new Resource() 可能发布不完整的对象
    // （对象字段写入 hb resource 赋值 不成立，可能看到半初始化对象）
}

class SafeLazyInit {
    private static volatile Resource resource;  // volatile 写屏障
    
    // volatile 写 resource = new Resource() 之前，
    // 所有字段的写操作都已完成（StoreStore 屏障），
    // 其他线程读到 resource 非null时，能看到完整初始化的对象
}
```

## 关联知识点
