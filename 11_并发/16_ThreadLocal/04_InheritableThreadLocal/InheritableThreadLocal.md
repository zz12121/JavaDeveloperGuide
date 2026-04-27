
# InheritableThreadLocal

## 核心结论

`InheritableThreadLocal` 继承自 `ThreadLocal`，新增了一个特性：**子线程自动继承父线程的 ThreadLocal 值**。拷贝发生在 `Thread` 构造函数的 `init()` 方法中，通过 `createInheritedMap()` 浅拷贝父线程的 `inheritableThreadLocals`。

## 深度解析

### 继承机制

```
父线程(main)                          子线程(new Thread)
┌────────────────────┐                ┌────────────────────┐
│ threadLocals       │                │ threadLocals       │
│ ┌────────────────┐ │                │ (空)               │
│ │ ThreadLocal→V  │ │                └────────────────────┘
│ └────────────────┘ │
│ inheritableThread  │    new Thread() │ inheritableThread  │
│ Locals             ├───────拷贝──────→│ Locals             │
│ ┌────────────────┐ │                │ ┌────────────────┐ │
│ │ ITL→"user-A"  │ │                │ │ ITL→"user-A"  │ │
│ └────────────────┘ │                │ └────────────────┘ │
└────────────────────┘                └────────────────────┘
         set("user-A")                      get() → "user-A" ✅
```

### 源码分析

```java
// Thread.init() — 拷贝发生的位置
private void init(ThreadGroup g, Runnable target, String name,
                  long stackSize, AccessControlContext acc,
                  boolean inheritThreadLocals) {
    // ...
    if (inheritThreadLocals && parent.inheritableThreadLocals != null)
        this.inheritableThreadLocals =
            ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
}

// createInheritedMap — 浅拷贝
static ThreadLocalMap createInheritedMap(ThreadLocalMap parentMap) {
    return new ThreadLocalMap(parentMap);
}

// ThreadLocalMap 拷贝构造
private ThreadLocalMap(ThreadLocalMap parentMap) {
    for (Entry e : parentMap.table) {
        if (e != null) {
            @SuppressWarnings("unchecked")
            ThreadLocal<Object> key = (ThreadLocal<Object>) e.get();
            if (key != null) {
                Object value = e.value;
                // 子线程的 Entry 引用同一个 key（ThreadLocal 实例）
                // value 是浅拷贝（引用同一个对象）
                childValue(value);  // 默认直接返回原 value（浅拷贝）
            }
        }
    }
}
```

### 与 ThreadLocal 的区别

| 特性 | ThreadLocal | InheritableThreadLocal |
|------|------------|----------------------|
| 数据存储 | `thread.threadLocals` | `thread.inheritableThreadLocals` |
| 线程间隔离 | 完全隔离 | 子线程继承父线程值 |
| 拷贝时机 | 无拷贝 | `new Thread()` 时拷贝 |
| 拷贝方式 | — | 浅拷贝（引用同一对象） |
| 线程池兼容 | 每次手动 set | ❌ 复用线程不会重新拷贝 |

### 自定义值处理

```java
// 重写 childValue() 自定义子线程的值
private static final InheritableThreadLocal<User> USER = 
    new InheritableThreadLocal<User>() {
        @Override
        protected User childValue(User parentValue) {
            // 深拷贝，避免父子线程共享同一对象
            return new User(parentValue.getName(), parentValue.getId());
        }
    };
```

### 适用场景

- 主线程设置用户上下文 → 子线程自动获取
- 日志 MDC 传递（子线程自动继承 traceId）
- 父子线程需要共享的配置信息

### 局限性

- **只支持创建子线程时拷贝**，线程池复用时不会重新拷贝
- **浅拷贝**，如果 value 是可变对象，父子线程共享同一引用
- **线程池场景需要额外方案**（见 [[Card_186_InheritableThreadLocal问题]]）

# InheritableThreadLocal问题

## 核心结论

`InheritableThreadLocal` 在**创建子线程时**拷贝父线程的值，但**线程池复用线程**时不会重新拷贝，导致子线程拿到的是首次创建时的旧值，产生数据错乱。

## 深度解析

### 问题复现

```java
public class ITLProblem {
    private static final InheritableThreadLocal<String> CONTEXT = 
        new InheritableThreadLocal<>();

    public static void main(String[] args) throws Exception {
        // 使用线程池（线程会复用）
        ExecutorService pool = Executors.newFixedThreadPool(1);

        // 第一次提交：设置 user=A
        CONTEXT.set("user=A");
        pool.submit(() -> {
            System.out.println("任务1: " + CONTEXT.get()); // user-A ✅
        }).get();

        // 第二次提交：设置 user=B，但线程被复用
        CONTEXT.set("user=B");
        pool.submit(() -> {
            System.out.println("任务2: " + CONTEXT.get()); // user-A ❌ 期望 user-B
        }).get();
    }
}
```

输出：

```
任务1: user-A
任务2: user-A    // ❌ 线程复用，值没有更新！
```

### 根因分析

```
父线程(main)          子线程(Thread-0, 线程池复用)
─────────────         ──────────────────────────
set("user=A")         init() 拷贝 → "user-A" ✅
                      submit() → get() = "user-A"
set("user=B")         线程已存在，不再 init()
                      submit() → get() = "user-A" ❌
```

**拷贝时机**：只在 `Thread` 构造函数中（`new Thread()` 时），`inheritableThreadLocals` 从父线程拷贝。

**线程池场景**：线程一旦创建就放入池中，后续提交任务复用已有线程，`init()` 不再执行，值永远是最初创建线程时的值。

### 与 ThreadLocal 对比

|特性|ThreadLocal|InheritableThreadLocal|
|---|---|---|
|线程隔离|✅ 完全隔离|✅ 创建时继承|
|线程池复用|每次需手动 set|❌ 复用时值不更新|
|适用场景|线程不复用 / 手动管理|一次性子线程|

## 解决方案

1. **每次提交任务前手动传递**（侵入性强）
2. **使用 TransmittableThreadLocal**（阿里 TTL 框架）
3. **使用装饰器包装 Runnable**（自定义传递逻辑）
## 关联知识点
