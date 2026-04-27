# CompletableFuture内部原理

## Q1：CompletableFuture 内部用什么保证线程安全？

**A**：使用 `volatile` + `CAS` 无锁算法：

```java
// result 是 volatile，保证可见性
volatile Object result;

// complete 使用 CAS，保证原子性
UNSAFE.compareAndSwapObject(this, RESULT, null, value);
```

**为什么不用 synchronized**：
- 状态转换极快（一两次 CAS）
- 大部分时间线程在等待 I/O，不竞争
- 无锁设计性能更好

---

## Q2：为什么 complete 只能成功一次？

**A**：`complete` 内部使用 `CAS(result, null, value)`：

```java
public boolean complete(T value) {
    return UNSAFE.compareAndSwapObject(this, RESULT, null, value);
}
```

- 第一次：result=null → CAS 成功 → 设置结果
- 第二次：result!=null → CAS 失败 → 返回 false，不覆盖

---

## Q3：join() 和 get() 有什么区别？

**A**：

| 区别 | join() | get() |
|------|--------|-------|
| 异常类型 | CompletionException（RuntimeException） | ExecutionException（受检异常） |
| 需处理 | 可选 | 必须 try-catch |
| 底层 | 相同，都调用 awaitDone() | 相同 |

```java
// join()：抛出 unchecked 异常
future.join(); // CompletionException

// get()：抛出受检异常
future.get(); // ExecutionException, InterruptedException
```

---

## Q4：CompletableFuture 的 Completion 是什么？

**A**：Completion 是回调任务的抽象基类：

```java
// 继承体系
Completion
├── UniCompletion (单个依赖)
│   ├── UniApply
│   ├── UniAccept
│   └── UniRun
├── BiCompletion (两个依赖)
│   ├── BiApply
│   ├── BiAccept
│   └── BiRun
└── ...
```

每种 Completion 对应一种回调类型（thenApply、thenAccept 等）。

---

## Q5：CompletableFuture 如何实现异步回调？

**A**：使用 Completion 栈 + 栈帧触发机制：

```java
// 1. 创建回调时，生成 Completion 节点
.thenApply(fn) → new UniApply(dst, this, fn) → push 到栈

// 2. 上游完成时，触发下游执行
postComplete() → 遍历 waiters 栈 → unpark 等待线程

// 3. 触发后执行 function
c.tryFire(mode) → uniApply(dst, fn, executor) → 执行回调
```

---

## Q6：为什么 thenApply 不能保证在同一线程执行？

**A**：取决于上游状态：

```java
// 情况1：上游已完成，立即触发，在当前线程执行
if (result != null) {
    uniApply(dst, fn, executor); // 同步执行
}

// 情况2：上游未完成，提交到线程池
else {
    Completion c = new UniApply(dst, fn);
    push(c);
    c.tryFire(ASYNC); // 异步执行
}
```

---

## Q7：AltResult 是什么？

**A**：异常结果的包装器：

```java
final class AltResult {
    final Throwable ex; // 异常对象

    // EXCEPTIONAL 是哨兵，表示"结果是异常"
    static final AltResult EXCEPTIONAL = new AltResult(null);
}

// 使用
result = new AltResult(new RuntimeException("error"));
```

---

## Q8：CompletableFuture 的状态转换是不可逆的吗？

**A**：是的，只能从未完成 → 已完成，不能回退：

```
UNCOMPLETED (0)
    │
    ├─ complete(value) ──► NORMAL (value)
    ├─ completeExceptionally ──► EXCEPTIONAL
    └─ cancel() ──► CANCELLED

一旦完成，永远保持该状态

obtrudeValue/obtrudeException 可以强制覆盖，但这是危险操作，已被标记 @Deprecated。

---

## Q9：CAS 失败后会怎么办？

**A**：自旋重试，直到成功：

```java
// Treiber Stack 的 pop 实现
do {
    h = head;           // 读取当前栈顶
    nh = h.next;        // 下一个节点
} while (!casHead(h, nh)); // CAS 失败则重试

// CompletableFuture 中的例子
do {
    c.next = waiters;
} while (!UNSAFE.compareAndSwapObject(this, WAITERS, c.next, c));
// CAS 失败说明其他线程改了栈顶，重试
```

---

## Q10：park 和 unpark 的原理是什么？

**A**：LockSupport 基于 Linux futex 实现：

```java
// park：线程进入 WAITING 状态，不消耗 CPU
LockSupport.park();           // 无限等待
LockSupport.parkNanos(nanos); // 等待指定时间

// unpark：唤醒指定线程
LockSupport.unpark(thread);

// 底层使用 futex 系统调用
// park = futex(WAIT, addr, timeout)
// unpark = futex(WAKE, addr)
```

---

## Q11：为什么用 Treiber Stack 而不是无锁队列？

**A**：Stack 更简单高效：

```java
// Stack 操作
push: CAS(head, old, new)  // 单次 CAS
pop:  CAS(head, old, new)  // 单次 CAS

// CompletableFuture 的特点：
// 1. Completion 执行一次后就移除
// 2. 大部分是顺序添加
// 3. LIFO 特性符合依赖关系
```

---

## Q12：postComplete 只唤醒 WaitNode，不触发 Completion？

**A**：是的，职责分离：

```
postComplete() 的职责：
  - unpark WaitNode（join/get 的线程）
  - 不触发 Completion

Completion 的触发：
  - 在下次 tryFire 时检查
  - source.result != null 时执行
```

---

## Q13：为什么 tryFire 只检查一次条件？

**A**：因为会多次调用：

1. push 后立即调用（可能上游已完成）
2. postComplete 后调用（线程池异步执行）
3. completion 时遍历调用

返回 null 表示"条件不满足"，下次满足时会再触发。

