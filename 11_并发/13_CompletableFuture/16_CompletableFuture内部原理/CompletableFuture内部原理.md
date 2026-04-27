# CompletableFuture 内部原理

## 核心结论

CompletableFuture 的核心是一个**状态机**，使用 `Unsafe.compareAndSetState()` 实现无锁并发控制：

```
┌────────────────────────────────────────────────────────────────────────────┐
│                         CompletableFuture 状态机                             │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│                              0 (UNCOMPLETED)                               │
│                                 │                                          │
│              ┌──────────────────┼──────────────────┐                       │
│              │                  │                  │                        │
│              ▼                  ▼                  ▼                        │
│         ┌────────┐        ┌────────────┐      ┌────────────┐               │
│         │  NORMAL │        │  EXCEPTIONAL │    │  CANCELLED │               │
│         │ (结果值) │        │  (异常结果)   │    │  (取消状态) │               │
│         └────────┘        └────────────┘      └────────────┘               │
│                                                                            │
│  状态编码：                                                                 │
│  - 0: UNCOMPLETED（未完成）                                                 │
│  - NEGATIVE: 表示结果（已完成）                                               │
│  - EXCEPTIONAL: 异常状态                                                     │
│  - CANCELLED: 取消状态                                                      │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

## 深度解析

### CompletableFuture 核心字段

```java
public class CompletableFuture<T> implements Future<T> {
    // 结果引用（volatile 保证可见性）
    volatile Object result;

    // 栈帧链表头（等待者的栈，用于单向链表）
    volatile WaitNode waiters;

    // 内部计数器（用于 allOf/anyOf 等）
    volatile int flags;
}
```

### result 的编码方式

```java
// result 的三种状态编码：

// 1. 正常结果：直接存储值
result = "hello";  // 正常完成，值为 "hello"

// 2. 异常结果：包装为 AltResult
result = new AltResult(new RuntimeException("error"));

// 3. AltResult.EXCEPTIONAL：表示"结果是异常"的哨兵
result = AltResult.EXCEPTIONAL;
```

### AltResult 异常包装

```java
// AltResult 是异常结果的包装器
final class AltResult {
    final Throwable ex; // 异常对象，EXCEPTIONAL 哨兵表示无异常

    AltResult(Throwable ex) {
        this.ex = ex;
    }
}

// 静态哨兵
static final AltResult EXCEPTIONAL = new AltResult(null);
```

### compareAndSetState：无锁状态转换

```java
// CompletableFuture 的核心CAS操作
private boolean internalComplete(Object r) {
    // UNCOMPLETED = 0
    // 使用 CAS 将状态从 UNCOMPLETED 改为结果 r
    return UNSAFE.compareAndSwapObject(this, RESULT, null, r);
}

// 实际使用示例
public boolean complete(T value) {
    // 只有 UNCOMPLETED 状态才能完成
    return boolean result = UNSAFE.compareAndSwapObject(
        this, RESULT, null, value  // CAS(null, value)
    );
}

public boolean completeExceptionally(Throwable ex) {
    return UNSAFE.compareAndSwapObject(
        this, RESULT, null, new AltResult(ex)  // CAS(null, AltResult)
    );
}
```

### 为什么用 CAS 而不是 synchronized？

```
┌────────────────────────────────────────────────────────────────────────────┐
│                          CAS vs Synchronized                              │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  synchronized：                                                             │
│  - 独占锁，阻塞其他线程                                                      │
│  - 线程上下文切换开销大                                                      │
│  - 适用于竞争激烈的场景                                                      │
│                                                                            │
│  CAS（compareAndSet）：                                                      │
│  - 无锁算法，非阻塞                                                         │
│  - 只有成功/失败，不阻塞线程                                                  │
│  - 适用于竞争不激烈的场景（CF 大部分时间在等待 I/O）                            │
│                                                                            │
│  CompletableFuture 选择 CAS 的原因：                                         │
│  1. 状态转换操作极快（只需一次 CAS）                                         │
│  2. 大部分时间在等待 I/O，线程不竞争                                         │
│  3. 无锁设计，性能更好                                                      │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

### 等待机制：WaitNode 栈

```java
// 等待者节点（单向链表）
static final class WaitNode implements ForkJoinPool.ManagedBlocker {
    volatile Thread thread;  // 等待线程
    WaitNode next;          // 栈的下一个节点

    // 栈结构：LIFO
    //      head
    //        │
    //        ▼
    //    [Node3] → [Node2] → [Node1] → null
}

// postComplete：完成时唤醒所有等待者
private void postComplete() {
    CompletableFuture<?> f = this;
    WaitNode h;
    while ((h = f.waiters) != null &&
           UNSAFE.compareAndSwapObject(f, WAITERS, h, h.next)) {
        // 唤醒等待线程
        h.thread.unpark();
    }
}
```

### 完整完成流程

```
┌────────────────────────────────────────────────────────────────────────────┐
│                         complete() 完整流程                                 │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  1. 外部调用 complete(value)                                                │
│         │                                                                   │
│         ▼                                                                   │
│  2. CAS(result, null, value)                                               │
│         │                                                                   │
│         ├── 失败（result != null）→ 返回 false，不重复完成                      │
│         │                                                                   │
│         └── 成功（result == null）→ 进入步骤3                                │
│                   │                                                         │
│                   ▼                                                         │
│  3. postComplete() 唤醒所有等待者                                            │
│         │                                                                   │
│         ▼                                                                   │
│  4. 遍历 waiters 栈，unpark 每个等待线程                                      │
│                                                                            │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │ 关键点：                                                           │  │
│  │ - CAS 保证只有第一个完成的线程能设置结果                               │  │
│  │ - 后续 complete 调用返回 false，不覆盖已有结果                          │  │
│  │ - waiters 栈保证所有等待者都能被唤醒                                   │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

### thenApply 的内部实现

```java
public <U> CompletableFuture<U> thenApply(Function<? super T, ? extends U> fn) {
    return uniApplyStage(null, fn);
}

private <V> CompletableFuture<V> uniApplyStage(Executor e, Function<? super T, ? extends V> f) {
    if (f == null) throw new NullPointerException();

    CompletableFuture<V> dst = new CompletableFuture<>();

    // 如果当前已完成，立即执行
    if (result != null) {
        // 异步或同步执行 function
        uniApply(dst, f, e);
    } else {
        // 未完成，创建一个依赖节点加入栈
        Completion c = new UniApply<>(e, dst, this, f);
        push(c);
        c.tryFire(SYNC); // 尝试触发
    }

    return dst;
}
```

### Completion 继承体系

```
┌────────────────────────────────────────────────────────────────────────────┐
│                         Completion 继承体系                                  │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│                         Completion (抽象)                                   │
│                              │                                              │
│         ┌───────────────────┼───────────────────┐                         │
│         │                   │                   │                         │
│    UniCompletion        BiCompletion        IndeterminateCompletion        │
│         │                   │                   │                         │
│   ┌─────┴─────┐       ┌─────┴─────┐                                      │
│   │           │       │           │                                      │
│ UniApply  UniAccept  BiApply  BiAccept  ...                                │
│   │           │       │           │                                      │
│ UniRun   UniRunAsync BiRun   BiRunAsync ...                                │
│   │           │       │           │                                      │
│ UniCompose       BiCompose     ...                                        │
│                                                                            │
│  每种 Completion 对应一种回调类型                                            │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

### join() / get() 的区别

```java
// join()：不抛出受检异常，直接抛出 unchecked 异常
public T join() {
    Object r;
    if ((r = result) == null) {
        // 包装为 RuntimeException
        throw uncheckedCompletionException(r);
    }
    return (T) r;
}

// get()：抛出受检异常，需要 try-catch
public T get() throws InterruptedException, ExecutionException {
    Object r;
    if ((r = result) == null) {
        awaitDone(...); // 等待完成
    }
    return reportGet(r);
}

// 内部都调用了 awaitDone()，只是异常处理不同
```

## 面试高频考点

### 考点1：为什么 complete 只能调用一次？

```java
CompletableFuture<String> cf = new CompletableFuture<>();

cf.complete("A"); // 返回 true，设置结果
cf.complete("B"); // 返回 false，不设置（已有结果）

cf.join(); // "A"，永远是 "A"
```

**答案**：
- `complete` 内部使用 `CAS(result, null, value)`
- 第一次调用 CAS 成功，设置结果
- 第二次调用 CAS 失败（result != null），返回 false
- 不可覆盖已存在的结果

### 考点2：join() 和 get() 的区别？

```java
// get() 抛出 ExecutionException（受检异常）
try {
    future.get();
} catch (ExecutionException | InterruptedException e) {
    // 需要处理两种异常
}

// join() 抛出 CompletionException（运行时异常）
try {
    future.join();
} catch (CompletionException e) {
    // 只需要处理运行时异常
}
```

### 考点3：CompletableFuture 是线程安全的吗？

**答案**：是的，使用 CAS + volatile 保证线程安全。

```java
// result 是 volatile，保证可见性
volatile Object result;

// complete 使用 CAS，保证原子性
UNSAFE.compareAndSwapObject(this, RESULT, null, value);
```

## 深度原理：Unsafe 与内存布局

### Unsafe 的使用

```java
// CompletableFuture 使用 Unsafe 直接操作内存
private static final sun.misc.Unsafe UNSAFE;

// 字段偏移量（通过反射计算）
private static final long RESULT = // result 字段在对象中的偏移量
    objectFieldOffset("result");
private static final long WAITERS = // waiters 字段偏移量
    objectFieldOffset("waiters");

// 计算字段偏移量
private static long objectFieldOffset(String fieldName) {
    try {
        return UNSAFE.objectFieldOffset(
            CompletableFuture.class.getDeclaredField(fieldName)
        );
    } catch (NoSuchFieldException e) {
        throw new Error(e);
    }
}
```

**为什么要用 Unsafe 而不用反射**：
- 反射有安全检查，性能差
- Unsafe 可以直接操作内存，无检查
- CAS 操作必须用 Unsafe

### 内存布局图

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    CompletableFuture 对象内存布局                              │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  对象头（Mark Word + Klass Pointer）                                         │
│         │                                                                  │
│         ▼                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  volatile Object result  │  volatile WaitNode waiters  │ flags     │   │
│  │       8 bytes            │       8 bytes              │  4 bytes   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                            │
│  RESULT 偏移量                                                             │
│      │                                                                    │
│      └──────────────────────────────────► 0x010（示例）                     │
│                                                                            │
│  WAITERS 偏移量                                                            │
│      │                                                                    │
│      └──────────────────────────────────► 0x018（示例）                     │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

### CAS 操作的具体实现

```java
// UNSAFE.compareAndSwapObject 签名
public final native boolean compareAndSwapObject(
    Object obj,     // 对象
    long offset,    // 字段偏移量
    Object expect,  // 期望值
    Object update   // 更新值
);

// complete 的 CAS 过程
public boolean complete(T value) {
    // 伪代码
    // if (this.result == null) {
    //     this.result = value;
    //     return true;
    // }
    // return false;

    // 实际等价于（但线程安全）：
    return UNSAFE.compareAndSwapObject(
        this,        // 在哪个对象上操作
        RESULT,      // 操作哪个字段（result）
        null,        // 期望当前值（必须是 null 才能成功）
        value        // 想要设置的新值
    );
}
```

### CAS 的三条总线操作

```
┌────────────────────────────────────────────────────────────────────────────┐
│                           CAS 硬件层面实现                                  │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  CPU 总线：                                                                │
│  ┌────────┐     ┌────────┐     ┌────────┐                                 │
│  │ CPU 1  │────▶│ BUS    │◀────│ CPU 2  │                                 │
│  └────────┘     └────────┘     └────────┘                                 │
│                      │                                                      │
│                      ▼                                                      │
│                 ┌────────┐                                                  │
│                 │  RAM   │                                                  │
│                 └────────┘                                                  │
│                                                                            │
│  CAS 执行过程：                                                             │
│  1. CPU1 读取内存到寄存器（result = null）                                  │
│  2. CPU1 比较寄存器值和内存值（null == null）→ 相等                          │
│  3. CPU1 写入新值（result = value）                                        │
│  4. 如果期间 CPU2 修改了内存，步骤2会发现不等，放弃重试                         │
│                                                                            │
│  ⚠️ ABA 问题：                                                             │
│  A → B → A 的情况可能通过 CAS，但状态实际已变化                               │
│  解决：用版本号（AtomicStampedReference）                                    │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

## 深度原理：Completion 栈实现

### Completion 的结构

```java
abstract static class Completion extends ForkJoinTask<Void> implements Runnable {
    volatile CompletableFuture<?> source;  // 上游 CF
    volatile CompletableFuture<?> dep;     // 依赖的 CF（结果写入这里）
    volatile int indexor;                  // 索引或状态

    // push 时设置
    Completion next;                       // 链表下一个节点

    // tryFire 模式
    static final int SYNC   = 0;  // 同步触发
    static final int ASYNC  = 1;  // 异步触发
    static final int NESTED = -1; // 嵌套（不重新提交）
}
```

### UniApply 的具体实现

```java
static final class UniApply<T, V> extends Completion {
    final Function<? super T, ? extends V> fn;
    final Executor executor;

    UniApply(Executor executor, CompletableFuture<V> dep,
             CompletableFuture<T> src, Function<? super T, ? extends V> fn) {
        this.executor = executor;
        this.dep = dep;
        this.source = src;
        this.fn = fn;
    }

    // 核心触发逻辑
    final CompletableFuture<V> tryFire(int mode) {
        CompletableFuture<T> a;
        CompletableFuture<V> c;
        if ((c = dep) != null && // 依赖 CF 存在
            c.result == null &&   // 依赖 CF 还未完成
            (a = source) != null && // 上游存在
            a.result != null) {   // 上游已完成 ← 关键条件
            // 从上游获取结果
            Object r = a.result;
            // 执行 function
            V v = applyUnionFn(r, fn);
            // 完成依赖 CF
            c.completeValue(v);
            return c; // 返回完成的 CF
        }
        return null; // 条件不满足，无法触发
    }

    private static <T, V> V applyUnionFn(Object r, Function fn) {
        // 处理异常情况
        if (r instanceof AltResult) {
            Throwable x = ((AltResult) r).ex;
            if (x != null) {
                throw new CompletionException(x);
            }
        }
        @SuppressWarnings("unchecked")
        T t = (T) r;
        return fn.apply(t);
    }
}
```

### push 的具体实现

```java
// 将 Completion 加入依赖 CF 的栈
final void push(Completion c) {
    // c 是新的 Completion
    // this 是依赖的 CompletableFuture

    if (c != null) {
        // 经典的 Treiber Stack 实现
        do {
            // 记录当前栈顶
            c.next = waiters;
        } while (!UNSAFE.compareAndSwapObject(
            this, WAITERS, c.next, c)); // CAS 新节点为栈顶
        // 如果 CAS 失败（其他线程先改了栈顶），重试
    }
}
```

### postComplete 的具体实现

```java
// CF 完成后，触发所有依赖
private void postComplete() {
    CompletableFuture<?> f = this;
    WaitNode h;

    // 遍历 waiters 栈（Treiber Stack 出栈）
    while ((h = f.waiters) != null && // 栈不为空
           UNSAFE.compareAndSwapObject(f, WAITERS, h, h.next)) {
        // CAS 成功，弹出一个节点
        // 唤醒等待线程
        h.thread.unpark();
    }

    // 注意：这里只唤醒 WaitNode，不触发 Completion
    // Completion 的触发是在依赖的 CF 完成后
}
```

## 深度原理：awaitDone 实现

### awaitDone 的核心逻辑

```java
// 等待 CF 完成
private Object awaitDone(WaitNode node, boolean timed, long nanos) {
    long deadline = timed ? System.nanoTime() + nanos : 0L;
    CompletableFuture<?> f = this;

    // 自旋 + LockSupport.park
    for (;;) {
        Object r = f.result;

        // 情况1：已完成
        if (r != null) {
            // 移除当前 WaitNode（如果有）
            if (node != null) {
                node.thread = null;
            }
            return r; // 返回结果
        }

        // 情况2：未完成，线程即将 park
        Thread t = Thread.currentThread();
        if (t.isInterrupted()) {
            // 被中断，移除 WaitNode
            if (node != null) {
                removeWaiter(node);
            }
            return f.result; // 可能是 null
        }

        // 情况3：park 之前先设置 WaitNode
        if (node == null) {
            node = new WaitNode();
        }
        node.thread = t;

        // 加入 waiters 栈
        if (f.waiters == null || !UNSAFE.compareAndSwapObject(f, WAITERS, node, node)) {
            // CAS 失败，栈被其他线程修改，继续自旋
        }

        // park
        if (timed) {
            LockSupport.parkNanos(this, nanos);
        } else {
            LockSupport.park(this);
        }

        // 醒来后继续自旋检查结果
    }
}
```

### WaitNode 与 Completion 的关系

```
┌────────────────────────────────────────────────────────────────────────────┐
│                        waiters vs completions                               │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  waiters 栈（WaitNode）：                                                   │
│  - 存储等待 join()/get() 的线程                                             │
│  - join() 时创建 WaitNode 加入栈                                            │
│  - CF 完成后 unpark 所有等待线程                                             │
│                                                                            │
│  completions 栈（Completion）：                                              │
│  - 存储依赖的回调任务（thenApply、thenAccept 等）                              │
│  - 上游 CF 完成后，Completion 尝试触发                                      │
│  - 触发成功则执行 function，失败则保持等待                                     │
│                                                                            │
│  两者都是 Treiber Stack，都用 CAS 保证线程安全                               │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

## 深度原理：thenApply 完整流程

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    thenApply(source, fn) 完整流程                            │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  线程A: [source CF] ──未完成──▶ thenApply(fn)                              │
│         │                                    │                              │
│         │                                    ▼                              │
│         │                         ┌──────────────────┐                     │
│         │                         │ 创建 UniApply    │                     │
│         │                         │  (Completion)    │                     │
│         │                         └────────┬─────────┘                     │
│         │                                  │                               │
│         ▼                                  ▼                               │
│  [push UniApply]                 [push(Completion)]                         │
│         │                                    │                              │
│         ▼                                    ▼                              │
│  [返回 dst CF]                    [加入 dst 的栈]                          │
│                                                                            │
│  线程A: [source CF] ──完成──▶ postComplete()                              │
│         │                           │                                       │
│         ▼                           ▼                                       │
│  unpark 所有 WaitNode        [遍历 completions 栈]                          │
│                                    │                                       │
│                                    ▼                                       │
│                           [tryFire(SYNC)]                                   │
│                                    │                                       │
│                                    ▼                                       │
│                      [读取 source.result]                                  │
│                                    │                                       │
│                                    ▼                                       │
│                      [执行 fn.apply(result)]                               │
│                                    │                                       │
│                                    ▼                                       │
│                      [dst.completeValue(v)]                                 │
│                                    │                                       │
│                                    ▼                                       │
│                      [dst.postComplete()]                                  │
│                                    │                                       │
│                                    ▼                                       │
│                         [唤醒 dst 的 WaitNode]                             │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

### thenApply 异步版本的区别

```java
// thenApply（同步版本）
public <U> CompletableFuture<U> thenApply(Function fn) {
    return uniApplyStage(null, fn);
}

// thenApplyAsync（异步版本）
public <U> CompletableFuture<U> thenApplyAsync(Function fn) {
    return uniApplyStage(defaultExecutor(), fn); // 使用默认 Executor
}

// thenApplyAsync(executor)（指定 Executor）
public <U> CompletableFuture<U> thenApplyAsync(Function fn, Executor executor) {
    return uniApplyStage(executor, fn);
}

// 区别在于 uniApplyStage 中
private <V> CompletableFuture<V> uniApplyStage(Executor e, Function fn) {
    CompletableFuture<V> dst = new CompletableFuture<>();

    if (result != null) {
        // 情况1：上游已完成
        if (e == null) {
            // 同步执行
            uniApply(dst, fn, e);
        } else {
            // 异步执行
            e.execute(() -> uniApply(dst, fn, null));
        }
    } else {
        // 情况2：上游未完成
        Completion c = new UniApply<>(e, dst, this, fn);
        push(c);
        c.tryFire(SYNC); // 同步尝试触发（可能上游在此期间完成了）
    }

    return dst;
}
```

## 面试高频追问

### 追问1：CAS 失败后怎么办？

**A**：自旋重试，直到成功：

```java
// Treiber Stack 的 pop 实现
do {
    h = head;      // 读取当前栈顶
    if (h == null) return null; // 栈空
    nh = h.next;   // 下一个节点
} while (!casHead(h, nh)); // CAS 替换，成功则返回，失败则重试

// 相当于：
// while (!CAS(head, h, h.next)) {
//     h = head; // 重新读取
// }
```

### 追问2：park 和 unpark 的原理？

**A**：LockSupport 的底层机制：

```java
// park：让线程进入等待状态，直到被 unpark 或中断
LockSupport.park();           // 无限等待
LockSupport.parkNanos(nanos); // 等待指定时间

// unpark：唤醒指定线程
LockSupport.unpark(thread);

// 底层：
// - 使用 Linux futex（快速用户态互斥）
// - park = futex(WAIT)
// - unpark = futex(WAKE)
```

### 追问3：为什么用 Treiber Stack 而不是队列？

**A**：性能考虑：

| 结构 | push | pop | 适用场景 |
|------|------|-----|---------|
| Treiber Stack | O(1) | O(1) | 单线程 push，多线程 pop |
| 无锁队列 | O(1) | O(1) | 多线程并发 |

```java
// Stack 的 push/pop 都是简单的 CAS
push: CAS(head, old, new)  // O(1)
pop:  CAS(head, old, new)  // O(1)

// CompletableFuture 的特点：
// - 回调是一次性的（执行后就从栈移除）
// - 大部分时间是顺序添加
// - Stack 的 LIFO 特性刚好符合
```

### 追问4：为什么 uniApply 的 tryFire 只检查一次？

```java
final CompletableFuture<V> tryFire(int mode) {
    // ⚠️ 只检查一次
    if ((c = dep) != null &&
        c.result == null &&  // 只检查一次
        (a = source) != null &&
        a.result != null) {  // 只检查一次
        // 执行...
    }
    return null; // 条件不满足，返回 null
}
```

**A**：因为 tryFire 在多种场景被调用：

1. **push 后立即 tryFire(SYNC)**：可能上游已完成，直接执行
2. **postComplete 后 tryFire(ASYNC)**：提交到线程池执行
3. **completion 完成时**：遍历栈时调用

如果 tryFire 返回 null，说明条件不满足：
- 上游还未完成 → 保持等待，由 postComplete 后续触发
- 依赖 CF 已完成 → 忽略（不会有这种情况）

### 追问5：postComplete 和 Completion 触发的时序？

```
时机1：complete() 被调用
    │
    ▼
complete(value) → CAS(result, null, value)
    │
    ▼
postComplete()
    │
    ├─ unpark WaitNode（join/get 的线程醒来）
    │
    └─ 触发 Completion？（Completion 不在 postComplete 中触发！）

时机2：Completion 的触发
    │
    ▼
Completion 是在依赖的 CF 完成后被触发
    │
    ▼
如何触发？通过 Completion 内部的检查：
    │
    ▼
tryFire(SYNC/ASYNC) 检查 source.result != null
    │
    ▼
如果上游已完成，执行 function，完成下游 CF
```

## 关联知识点

- [[Cancellation专题|Cancellation 机制]]
- [[exceptionally和handle|exceptionally 与 handle]]
- [[虚拟线程与CompletableFuture|虚拟线程与 CompletableFuture]]
