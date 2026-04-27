
# Phaser

## Q1：Phaser 和 CyclicBarrier 有什么区别？

**A**：

- **动态参与者**：CyclicBarrier 的 parties 固定不变，Phaser 支持运行时 `register()` / `arriveAndDeregister()` 动态增减
- **自动阶段推进**：Phaser 自动维护阶段号（phase），无需手动管理
- **终止控制**：通过覆写 `onAdvance()` 自定义终止条件
- **层级支持**：Phaser 可以构造子 Phaser 形成树状层级

Phaser 是 CyclicBarrier 的超集，功能更强但更复杂。

---

## Q2：Phaser 的 arriveAndAwaitAdvance 和 CyclicBarrier 的 await 有什么区别？

**A**：功能类似（到达并等待），但 Phaser 的 `arriveAndAwaitAdvance` 返回到达的阶段号，可以用于判断是否推进了新阶段。

此外，Phaser 还提供 `arrive()`（到达但不等待），这是 CyclicBarrier 没有的功能。

---

## Q3：Phaser 在实际项目中有哪些使用场景？

**A**：实际项目中直接使用较少，典型场景：

1. **多阶段并行计算**：如 MapReduce 的多个阶段，每阶段所有线程对齐
2. **动态任务编排**：任务执行过程中需要动态添加/移除参与者
3. **模拟仿真**：离散事件仿真中按时间步同步

大多数场景下 CyclicBarrier 已经够用，知道 Phaser 的概念和优势即可。

---
```java
Phaser phaser = new Phaser(3);  // 3 个参与者

// 动态注册/注销
phaser.register();  // 新增参与者
phaser.arriveAndDeregister();  // 完成+注销

// 到达并等待（类似 CyclicBarrier.await）
phaser.arriveAndAwaitAdvance();  // 返回阶段号

// 自定义终止条件（3 轮后终止）
Phaser phaser = new Phaser(3) {
    @Override
    protected boolean onAdvance(int phase, int registeredParties) {
        return phase >= 2;  // phase 0,1,2 共 3 轮，phase=2 后终止
    }
};
```


