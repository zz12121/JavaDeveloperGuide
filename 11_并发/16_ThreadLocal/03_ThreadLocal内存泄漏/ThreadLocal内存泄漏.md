
# ThreadLocal内存泄漏

## 核心结论

`ThreadLocal` 内存泄漏的根本原因是：`ThreadLocalMap.Entry` 的 key 是弱引用（可被 GC），但 **value 是强引用（不可被 GC）**。当 key 被 GC 回收后，value 仍被 Entry 强引用，如果线程不消亡（线程池复用），value 就永远无法回收。

## 深度解析

### 引用链分析

```
情况一：ThreadLocal 外部强引用存在（正常使用）
┌──────────┐      强引用     ┌──────────────────────────────┐
│ ThreadLocal├───────────────│ Entry(key=ThreadLocal,value=V)│
│ 实例       │               └──────────────────────────────┘
└──────────┘
                 ✅ key 和 value 都可达，不会被回收

情况二：ThreadLocal 外部强引用消失（泄漏风险）
               GC 回收 key
┌──────────┐                 ┌──────────────────────────────┐
│ ThreadLocal│    ~~弱引用~~→ │ Entry(key=null,   value=V)   │
│ (已被GC)   │               └──────────────────────────────┘
└──────────┘                       ↑
                                   │ 强引用
                              Thread.threadLocals → ThreadLocalMap → Entry → value
                              ↑ 线程不消亡，整条引用链一直存在
                              ❌ value 无法回收 → 内存泄漏
```

### 泄漏触发条件

1. ThreadLocal 外部强引用消失（如局部变量出作用域）
2. ThreadLocal 被 GC 回收（key 变为 null）
3. 当前线程**仍在运行**（线程池复用线程不会消亡）
4. 没有手动调用 `remove()`

```java
public void process() {
    ThreadLocal<byte[]> data = new ThreadLocal<>();
    data.set(new byte[1024 * 1024 * 10]);  // 10MB
    // 方法结束，data 强引用消失
    // 但线程还在（线程池），value 的 10MB 永远无法回收！
}
```

### JDK 的自动清理机制

`ThreadLocalMap` 在调用 `set()`、`get()`、`remove()` 时会触发清理，共有两种清理方式：

#### 探测式清理（expungeStaleEntry）

从一个 stale slot 开始，**线性向后扫描**到 null 槽位为止：
- 遇到 `key == null` 的 Entry → 清空 value 和槽位，断开强引用
- 遇到 `key != null` 的 Entry → 重新 rehash 到正确位置（防止线性探测链断裂）

**触发时机**：`get()` 探测时遇到 stale entry、`remove()` 删除后、`set()` 的 `replaceStaleEntry` 内部

#### 启发式清理（cleanSomeSlots）

在 `set()` 插入新 entry 之后调用，以 **log2(n)** 次数跳跃扫描：
- 发现 stale entry → 调用 `expungeStaleEntry` 精确清理，并重置扫描计数
- 目的：以最小代价顺带清理，不阻塞正常操作

```java
// ThreadLocalMap.set() 中的清理触发
private void set(ThreadLocal<?> key, Object value) {
    // ...线性探测
    if (e.get() == null) {
        // 遇到 key==null 的陈旧 Entry → replaceStaleEntry 内调用 expungeStaleEntry
        replaceStaleEntry(key, value, i);
        return;
    }
    // ...插入成功后
    cleanSomeSlots(i, sz);  // 启发式清理
}
```

**局限性**：如果线程池中的线程执行完任务后空闲，不再调用 `set/get/remove`，陈旧 Entry 永远不会被自动清理，必须手动 `remove()`。

### 线程池场景的严重性

```
FixedThreadPool(4) 中 4 个线程长期存活：

任务1: ThreadLocal_A.set(10MB) → 任务结束，忘记 remove
       → ThreadLocal_A 被 GC，key=null，value=10MB 残留

任务2: ThreadLocal_B.set(10MB) → 任务结束，忘记 remove
       → ThreadLocal_B 被 GC，key=null，value=10MB 残留

... 重复执行 N 次后，每个线程的 ThreadLocalMap 中积累大量陈旧 Entry
→ 内存持续增长 → OOM
```

## 解决方案

```java
// 标准使用模式
try {
    threadLocal.set(value);
    // 业务逻辑
} finally {
    threadLocal.remove();  // ✅ 唯一可靠的方式
}
```

## 关联知识点
