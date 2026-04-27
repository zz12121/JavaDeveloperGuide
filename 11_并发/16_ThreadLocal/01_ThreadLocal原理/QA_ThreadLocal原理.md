
# ThreadLocal原理

## Q1：ThreadLocal 的底层实现原理是什么？

**A**：

每个 `Thread` 对象内部持有一个 `ThreadLocalMap` 类型的字段 `threadLocals`。`ThreadLocalMap` 的 key 是 `ThreadLocal` 实例（弱引用），value 是实际存储的值（强引用）。

当调用 `ThreadLocal.get()` 时：
1. 获取当前线程 `Thread.currentThread()`
2. 取出线程的 `ThreadLocalMap`
3. 以 `ThreadLocal` 实例为 key 在 map 中查找对应的 Entry
4. 返回 Entry 的 value

不同线程有各自的 `ThreadLocalMap`，同一个 `ThreadLocal` 实例作为 key，但存取的是各自 map 中的 value，因此实现了线程隔离。

---

## Q2：ThreadLocalMap 的 hash 冲突怎么解决？

**A**：

`ThreadLocalMap` 使用**开放寻址法（线性探测）**解决冲突：

- 计算 `key.threadLocalHashCode & (table.length - 1)` 定位初始位置
- 如果该位置已被占用且 key 不匹配，就向后逐个查找（i+1, i+2...），直到找到空位或匹配的 key

这与 `HashMap` 的链表/红黑树不同。`ThreadLocalMap` 选择开放寻址法是因为：
1. `ThreadLocal` 数量通常很少，冲突概率低
2. 探测效率高，不需要额外的链表节点对象开销
3. 内存更紧凑

---

## Q3：ThreadLocalMap 的 hash 值为什么用 0x61c88647？

**A**：

`0x61c88647` 是黄金比例（`(√5 - 1) / 2`）乘以 `2^32` 的整数近似值，被称为**斐波那契散列**。

使用这个魔数的好处：
- hash 值在 2 的幂次方大小的数组中分布极其均匀
- 相邻 ThreadLocal 的 hash 值分散在数组各段，减少冲突

每次创建 `ThreadLocal` 时，`threadLocalHashCode` 递增 `0x61c88647`，确保不同 ThreadLocal 实例的 hash 值分布均匀。

---

## Q4：为什么 ThreadLocalMap 的 key 设计为弱引用？

**A**：

如果 key 是强引用，当外部对 `ThreadLocal` 的引用消失后（如方法结束，局部变量被回收），由于 `ThreadLocalMap` 中还有强引用指向它，`ThreadLocal` 对象无法被 GC 回收。

设计为弱引用后：
- 外部强引用消失 → GC 回收 key（ThreadLocal 对象）
- Entry 的 `key == null`，触发清理逻辑

但 **value 仍然是强引用**，即使 key 被 GC 回收，value 还在，这是内存泄漏的根因（见 [[Card_183_ThreadLocal内存泄漏]]）。

---
```java
// ThreadLocal 原理：每个线程有自己的 ThreadLocalMap
ThreadLocal<User> userHolder = new ThreadLocal<>();

// set：当前线程的 ThreadLocalMap[key=ThreadLocal, value=User]
userHolder.set(new User("Alice"));

// get：从当前线程的 ThreadLocalMap 取值
User user = userHolder.get();

// 内存结构：
// Thread → ThreadLocalMap → Entry[] → Entry(ThreadLocal[weak], User[strong])
// key 是弱引用：ThreadLocal 外部引用消失后可被 GC
// value 是强引用：必须手动 remove()，否则内存泄漏
```

