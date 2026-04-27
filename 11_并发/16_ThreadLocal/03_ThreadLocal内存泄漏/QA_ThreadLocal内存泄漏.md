
# ThreadLocal内存泄漏

## Q1：ThreadLocal 内存泄漏的根本原因是什么？

**A**：

内存泄漏的根本原因是 `ThreadLocalMap.Entry` 的 **key 是弱引用，value 是强引用**，两者生命周期不一致。

**引用链分析**：

```
正常情况：
ThreadLocal变量(强引用) → ThreadLocal实例 ← key(弱引用) Entry → value(强引用)

泄漏情况（ThreadLocal变量置null后）：
ThreadLocal变量 → null
                   ThreadLocal实例 ← key(弱引用，GC后变null)
Thread → ThreadLocalMap → Entry(key=null, value=??) ← value(强引用，无法回收！)
```

**关键路径**：线程存活 → ThreadLocalMap 存活 → Entry 存活 → value 强引用 → value 无法回收

触发泄漏的4个条件必须同时满足：
1. ThreadLocal 外部强引用消失（如局部变量出了作用域）
2. GC 执行，key（弱引用）被回收，变为 null
3. 当前线程仍在运行（线程池复用，线程不死亡）
4. 没有手动调用 `remove()`

---

## Q2：expungeStaleEntry 是什么？它做了什么？（探测式清理）

**A**：

`expungeStaleEntry(int staleSlot)` 是 `ThreadLocalMap` 的**探测式清理**方法，从一个 stale slot（key=null 的槽位）开始，**线性向后扫描**直到遇到 `null` 槽位为止。

**清理过程**：

```java
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;
    
    // 第一步：清理起始 stale slot
    tab[staleSlot].value = null;  // 断开 value 强引用
    tab[staleSlot] = null;        // 清空槽位，GC 可回收
    size--;
    
    // 第二步：继续线性扫描后续槽位
    Entry e;
    int i;
    for (i = nextIndex(staleSlot, len); (e = tab[i]) != null; i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
        if (k == null) {
            // 发现另一个 stale entry → 也清理掉
            e.value = null;
            tab[i] = null;
            size--;
        } else {
            // key 还在 → rehash 到正确位置（防止线性探测链断裂）
            int h = k.threadLocalHashCode & (len - 1);
            if (h != i) {
                tab[i] = null;
                while (tab[h] != null) h = nextIndex(h, len);
                tab[h] = e;
            }
        }
    }
    return i;  // 返回到达的 null 槽位索引
}
```

**两个关键操作**：
- **清理 stale entry**：`value = null` + `tab[i] = null`，断开强引用让 GC 回收
- **rehash 非 stale entry**：将移位了的 entry 搬回正确位置，防止后续 `get()` 找不到

---

## Q3：cleanSomeSlots 是什么？和 expungeStaleEntry 有什么区别？（启发式清理）

**A**：

`cleanSomeSlots` 是**启发式清理**方法，在 `set()` 插入新 entry 之后被调用，以**对数次数**（`log2(n)` 次）扫描槽位，用最小代价顺带清理 stale entry。

```java
private boolean cleanSomeSlots(int i, int n) {
    boolean removed = false;
    Entry[] tab = table;
    int len = tab.length;
    do {
        i = nextIndex(i, len);
        Entry e = tab[i];
        if (e != null && e.get() == null) {
            n = len;  // 发现 stale entry → 重置扫描次数，继续扫更多
            removed = true;
            i = expungeStaleEntry(i);  // 调用探测式清理
        }
    } while ((n >>>= 1) != 0);  // n 每次右移1位（log2 次）
    return removed;
}
```

**两种清理的核心区别**：

| 维度 | expungeStaleEntry（探测式） | cleanSomeSlots（启发式） |
|------|--------------------------|------------------------|
| 触发方式 | 被动触发（遇到 stale entry） | 主动调用（set 后顺带） |
| 扫描范围 | 从 stale slot 到下一个 null | log2(n) 个槽位 |
| 扫描策略 | 线性连续扫描 | 跳跃式（指数递减） |
| 调用关系 | 被 cleanSomeSlots 调用 | 调用 expungeStaleEntry |
| 目的 | 精确清理一段连续区域 | 以小代价覆盖更大范围 |
| 触发时机 | get() 探测miss时、remove() | set() 插入新entry后 |

**通俗理解**：
- `expungeStaleEntry`：地毯式清扫（某一段连续的房间彻底清干净）
- `cleanSomeSlots`：抽查巡逻（每隔几个房间扫一下，代价低，覆盖广）

---

## Q4：set()、get()、remove() 分别在什么时机触发清理？

**A**：

**set() 触发两次清理**：
```
set(key, value) 流程：
1. 线性探测，若遇到 stale entry（key=null）：
   → 调用 replaceStaleEntry()
     → 内部调用 expungeStaleEntry()（探测式清理一段区域）
     → 内部调用 cleanSomeSlots()（启发式清理更多区域）
2. 插入成功后：
   → 调用 cleanSomeSlots()（启发式清理）
3. size >= threshold → rehash()
   → rehash 内调用 expungeStaleEntries()（全量清理所有 stale entry）
```

**get() 触发一次清理**：
```
get(key) 流程：
1. 线性探测，若遇到 stale entry（key=null）：
   → 调用 expungeStaleEntry()（探测式清理）
2. 遇到 null 槽位 → 停止，返回 null
```

**remove() 触发一次清理**：
```
remove(key) 流程：
1. 找到 key 对应的 Entry
2. 调用 entry.clear()（WeakReference.clear()，将 key 置 null）
3. 立即调用 expungeStaleEntry()（清理 value 并修复后续 rehash 链）
```

**总结**：
- `remove()` 是最彻底的清理，能确保 value 立即被回收
- `set()/get()` 的清理是"顺带"的，不保证清理所有 stale entry

---

## Q5：为什么 JDK 的自动清理机制不可靠？什么情况下清理不会触发？

**A**：

**不可靠的根本原因**：`expungeStaleEntry` 和 `cleanSomeSlots` 只在 `set()`、`get()`、`remove()` 方法被调用时才会触发，是**被动式清理**。

**清理不会触发的场景**：

```
场景一：线程空闲（没有任务提交）
  - 线程池中的线程等待新任务
  - 没有任何 ThreadLocal 的 set/get/remove 调用
  - ThreadLocalMap 中的 stale entry 永远不会被自动清理

场景二：使用其他 ThreadLocal 而不使用泄漏的那个
  - 只调用了 threadLocalB.get()，不会触发 threadLocalA 对应的清理
  - 探测式清理只能清理"路径上"遇到的 stale entry

场景三：单次 set/get 之间 stale entry 很多
  - cleanSomeSlots 只扫描 log2(n) 次，不保证清理所有
  - 如果 stale entry 分散在数组各处，只清理了其中一部分
```

**实际危害**：
```java
// 线程池场景：线程长期存活，自动清理永远不触发
ThreadPoolExecutor pool = new ThreadPoolExecutor(4, 4, 0, ...);

// 每个任务结束后不 remove
pool.submit(() -> {
    CONTEXT.set(new byte[10 * 1024 * 1024]); // 10MB
    // 任务完成，线程归池，不再调用 set/get/remove
    // → stale entry 永远不会被清理
});
```

**结论**：JDK 的自动清理是"锦上添花"，不能替代手动 `remove()`。

---

## Q6：线程池场景下内存泄漏为何比普通线程更严重？

**A**：

**普通线程**：线程执行完毕后死亡，`Thread` 对象被 GC，其持有的 `ThreadLocalMap` 自动回收，不会泄漏。

**线程池线程**：线程长期存活（复用），`threadLocals` 字段一直存在，其中的 stale entry 也一直存在。

**叠加效应（最严重情况）**：

```
线程池 FixedThreadPool(4)，4个线程长期存活：

每个线程每轮任务积累：
  - 任务1: CONTEXT_A.set(10MB) → key GC 后变 null，value=10MB 残留
  - 任务2: CONTEXT_B.set(10MB) → key GC 后变 null，value=10MB 残留
  - 任务3: CONTEXT_C.set(10MB) → key GC 后变 null，value=10MB 残留

4个线程 × 每轮3个10MB残留 × N轮 = 120MB × N → OOM
```

**内存增长模式**：
- 普通线程：每次执行完毕后内存释放，波浪形曲线
- 线程池：内存只增不减（锯齿形曲线，最终触碰 JVM 堆上限）

**正确做法**：
```java
pool.submit(() -> {
    try {
        CONTEXT.set(value);
        doBusiness();
    } finally {
        CONTEXT.remove();  // 确保每次任务结束都清理
    }
});
```

---
