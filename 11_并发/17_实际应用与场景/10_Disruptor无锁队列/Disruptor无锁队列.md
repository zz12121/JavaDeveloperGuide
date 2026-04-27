
# Disruptor无锁队列

## 先说结论

Disruptor 是 LMAX 交易所开源的超高性能无锁队列，基于**环形缓冲区（Ring Buffer）** 实现。核心思想是**预分配内存 + 序号（Sequence）机制代替锁 + 缓存行填充消除伪共享**。LMAX 论文数据：单线程每秒可处理 **600 万+** 订单，比 ArrayBlockingQueue 快一个数量级。

## 深度解析

### 核心组件

```
Disruptor 架构：
┌─────────────────────────────────────────────────┐
│  Producer                                       │
│    │                                            │
│    ▼                                            │
│ ┌─────────────┐    ┌──────────────────────────┐ │
│ │ Ring Buffer  │───▶│ Event Processor (Consumer)│ │
│ │ (环形缓冲区) │    │  ┌─────┐ ┌─────┐ ┌─────┐│ │
│ │             │    │  │ C1  │ │ C2  │ │ C3  ││ │
│ │ [0][1][2].. │    │  └──┬──┘ └──┬──┘ └──┬──┘│ │
│ │             │    │     └───────┼───────┘   │ │
│ └─────────────┘    │             ▼           │ │
│                    │  ┌──────────────────┐   │ │
│                    │  │ Sequence Barrier  │   │ │
│                    │  │ (序号屏障/依赖管理) │  │ │
│                    │  └──────────────────┘   │ │
│                    └──────────────────────────┘ │
└─────────────────────────────────────────────────┘
```

| 组件 | 职责 |
|------|------|
| **Ring Buffer** | 环形数组，存储事件数据，预分配固定大小 |
| **Sequence** | 递增序号，无锁原子递增，替代锁机制 |
| **Sequence Barrier** | 协调消费者依赖关系，消费者等待前置消费者处理完成 |
| **Event** | 数据载体，存储在 Ring Buffer 的槽位中 |
| **EventProcessor** | 事件处理器，循环读取并处理事件 |
| **Producer** | 生产者，通过 Sequence 发布事件 |

### 为什么快

| 优化点 | 原理 | 效果 |
|--------|------|------|
| **预分配内存** | Ring Buffer 预先分配所有槽位，避免运行时 GC | 消除 GC 停顿 |
| **无锁设计** | 用 CAS 更新 Sequence 代替 synchronized/Lock | 消除锁竞争和上下文切换 |
| **缓存行填充** | Sequence 对象填充到 64 字节缓存行，避免伪共享 | 消除 CPU 缓存一致性流量 |
| **批量处理** | 消费者可批量处理多个事件，减少分支预测失败 | 提高 CPU 流水线效率 |
| **环形数组** | 数组下标取模，内存连续，CPU 缓存友好 | 提高缓存命中率 |

### Ring Buffer 原理

```
环形数组（大小=8）：
  published: 12          // 已发布到序号12
  consumer:  10          // 消费者已处理到序号10

  [0][1][2][3][4][5][6][7]
         ↑ Published = 12, actual slot = 12 % 8 = 4
  Slot 4,5,6,7,0,1,2,3 → 12,13,14,15,16,17,18,19
```

### 消除伪共享

```java
// LMAX 的缓存行填充方案
class LhsPadding {
    volatile long p1, p2, p3, p4, p5, p6, p7;
}

class Value extends LhsPadding {
    volatile long value;
}

class RhsPadding extends Value {
    volatile long p8, p9, p10, p11, p12, p13, p14;
}

// Sequence 继承 RhsPadding，确保 value 独占一个 64 字节缓存行
public class Sequence extends RhsPadding {
    // value 在缓存行中间，前后各 7 个 long（56字节）+ 8字节 = 64字节
}
```

## 代码示例

```java
// 1. 定义事件
public class TradeEvent {
    private long id;
    private String symbol;
    private double price;
    private int quantity;
    // getter/setter
}

// 2. 定义事件工厂
public class TradeEventFactory implements EventFactory<TradeEvent> {
    public TradeEvent newInstance() { return new TradeEvent(); }
}

// 3. 定义消费者
public class TradeEventHandler implements EventHandler<TradeEvent> {
    @Override
    public void onEvent(TradeEvent event, long sequence, boolean endOfBatch) {
        System.out.printf("处理订单: id=%d, symbol=%s, price=%.2f%n",
            event.getId(), event.getSymbol(), event.getPrice());
    }
}

// 4. 启动 Disruptor
Disruptor<TradeEvent> disruptor = new Disruptor<>(
    new TradeEventFactory(),
    1024,                                    // Ring Buffer 大小（必须是 2^n）
    Executors.defaultThreadFactory(),
    ProducerType.MULTI,                      // 多生产者
    new BlockingWaitStrategy()               // 等待策略
);
disruptor.handleEventsWith(new TradeEventHandler());
disruptor.start();

// 5. 发布事件
RingBuffer<TradeEvent> ringBuffer = disruptor.getRingBuffer();
long sequence = ringBuffer.next();          // 获取下一个可用序号
try {
    TradeEvent event = ringBuffer.get(sequence);
    event.setId(1L);
    event.setSymbol("AAPL");
    event.setPrice(175.50);
} finally {
    ringBuffer.publish(sequence);           // 发布事件
}
```

### 等待策略选择

| 等待策略 | 性能 | CPU 占用 | 适用场景 |
|---------|------|---------|---------|
| BlockingWaitStrategy | 最低 | 最低 | 低吞吐、低延迟要求 |
| SleepingWaitStrategy | 中等 | 低 | 平衡吞吐和CPU |
| YieldingWaitStrategy | 高 | 高 | 高吞吐、相同消费者线程 |
| BusySpinWaitStrategy | 最高 | 100% | 极低延迟、CPU 富余 |

## 易错点/踩坑

- ❌ Ring Buffer 大小设为非 2 的幂次方
- ✅ 大小必须是 2^n（用位运算代替取模）
- ❌ 多生产者场景使用 ProducerType.SINGLE
- ✅ 多生产者必须用 ProducerType.MULTI
- ❌ 忘记调用 ringBuffer.publish(sequence)
- ✅ 必须在 finally 块中 publish

