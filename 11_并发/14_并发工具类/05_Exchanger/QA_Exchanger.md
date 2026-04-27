
# Exchanger

## Q1：Exchanger 是做什么的？

**A**：Exchanger 允许两个线程在一个**同步点**交换数据。线程 A 调用 `exchange(dataA)` 阻塞等待，线程 B 调用 `exchange(dataB)` 后，A 收到 dataB，B 收到 dataA，双方都继续执行。

---

## Q2：Exchanger 的底层实现用了 AQS 吗？

**A**：没有。Exchanger 基于无锁算法实现：

- 低竞争时使用 **Slot（单槽）**，通过 CAS 占位
- 高竞争时切换到 **Arena（多槽数组）**，减少 CAS 冲突
- 不依赖 AQS，不使用 Lock/Condition

---

## Q3：Exchanger 有哪些使用场景？

**A**：比较小众，常见场景：

1. **遗传算法**：两个线程交换各自的最优解，交叉后产生更优解
2. **数据校对**：快速算法和精确算法分别处理，交换结果对比验证
3. **缓冲区交换**：双缓冲区技术中，填充线程和消费线程交换缓冲区

实际开发中用得不多，知道概念即可。

---
```java
Exchanger<String> exchanger = new Exchanger<>();

// 线程A：交换数据
new Thread(() -> {
    try {
        String fromB = exchanger.exchange("来自A的数据");
        System.out.println("A 收到: " + fromB);
    } catch (InterruptedException e) {}
}).start();

// 线程B：交换数据
new Thread(() -> {
    try {
        String fromA = exchanger.exchange("来自B的数据");
        System.out.println("B 收到: " + fromA);
    } catch (InterruptedException e) {}
}).start();
// A 收到: 来自B的数据
// B 收到: 来自A的数据
```
