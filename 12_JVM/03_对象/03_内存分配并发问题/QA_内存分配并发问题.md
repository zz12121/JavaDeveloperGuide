
# 内存分配并发问题

## Q1：内存分配为什么会存在并发问题？

**A**：多线程同时分配内存时，如果只有一个指针指向下一个可用内存位置：
- 线程 A 和 B 同时读取指针位置
- 线程 A 先更新指针
- 线程 B 基于旧指针更新（覆盖线程 A 的对象）

这是典型的并发竞争问题。

---

## Q2：JVM 如何解决并发分配问题？

**A**：两种解决方案：
1. **CAS + 失败重试**：乐观锁，失败重试
2. **TLAB（Thread Local Allocation Buffer）**：线程私有缓冲区

---

## Q3：CAS + 失败重试的原理？

**A**：
```java
// 伪代码
while (true) {
    int current = memoryPointer;
    int next = current + objectSize;
    // CAS 操作
    if (CAS(current, next)) {
        return; // 分配成功
    }
    // 分配失败，重试
}
```
- 高并发时重试次数多，CPU 开销大
- 适合分配不频繁的场景

---

## Q4：TLAB 为什么能解决并发问题？

**A**：TLAB 让每个线程在 Eden 区预分配一块私有内存：
- 线程只在自己的 TLAB 中分配
- 不需要任何同步操作
- 分配速度极快

---

## Q5：TLAB 分配失败怎么办？

**A**：TLAB 分配失败的处理：
1. **重新分配 TLAB**：线程申请新的 TLAB
2. **回退到 CAS**：TLAB 分配失败时使用同步方式
3. **参数控制**：`TLABRefillWasteFraction` 控制何时放弃 TLAB

---

## Q6：TLAB 大小的设置建议？

**A**：
- **太小**：频繁分配/重试 TLAB，开销大
- **太大**：浪费 Eden 区空间
- **默认**：约 1% Eden 区大小
- **一般不需要调整**：JVM 自动优化

---

## Q7：哪些对象不使用 TLAB 分配？

**A**：以下情况可能不使用 TLAB：
- **大对象**：超过 TLAB 大小的对象
- **TLAB 剩余不足**：无法容纳整个对象
- **TLAB 禁用**：`-XX:-UseTLAB`

---
