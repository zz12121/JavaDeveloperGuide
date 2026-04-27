
# Exchanger

## 核心结论

Exchanger 是一个线程间**交换数据**的工具类。两个线程在交换点通过 `exchange()` 方法互相传递数据，线程会阻塞直到配对线程也到达交换点。

## 深度解析

### 核心机制

```
Thread-A: exchange(dataA) → 阻塞等待
Thread-B: exchange(dataB) → 配对成功 → A 收到 dataB，B 收到 dataA
```

### 关键方法

| 方法 | 说明 |
|------|------|
| `exchange(V x)` | 交换数据，阻塞等待配对 |
| `exchange(V x, timeout, unit)` | 带超时的交换 |

### 内部实现

- 基于无锁算法（Slot + Arena），不使用 AQS
- 单槽（Slot）用于低竞争，多槽（Arena）用于高竞争
- 线程到达后通过 CAS 占位，配对线程到达时交换数据并唤醒

## 代码示例

### 基本交换

```java
Exchanger<String> exchanger = new Exchanger<>();

new Thread(() -> {
    try {
        String fromOther = exchanger.exchange("Hello from A");
        System.out.println("A received: " + fromOther);
    } catch (InterruptedException e) { }
}, "Thread-A").start();

new Thread(() -> {
    try {
        String fromOther = exchanger.exchange("Hello from B");
        System.out.println("B received: " + fromOther);
    } catch (InterruptedException e) { }
}, "Thread-B").start();
```

### 实际场景：遗传算法

```java
// 两个线程交换各自的最优解
Exchanger<Solution> exchanger = new Exchanger<>();

// Thread-1: 局部搜索
Solution localBest1 = search();
Solution otherBest = exchanger.exchange(localBest1);
Solution combined = merge(localBest1, otherBest);

// Thread-2: 同样进行
Solution localBest2 = search();
Solution otherBest = exchanger.exchange(localBest2);
```

### 实际场景：数据校对

```java
// 两个线程分别用不同算法处理同一批数据，交换结果进行对比
Exchanger<List<Result>> exchanger = new Exchanger<>();

Thread fast = new Thread(() -> {
    List<Result> fastResult = fastProcess(data);
    List<Result> accurateResult = exchanger.exchange(fastResult);
    compare(fastResult, accurateResult);
});

Thread accurate = new Thread(() -> {
    List<Result> accurateResult = accurateProcess(data);
    List<Result> fastResult = exchanger.exchange(accurateResult);
    compare(fastResult, accurateResult);
});
```

## 注意事项

- **只能配对两个线程**，如果有奇数个线程调用 exchange，多余的会永远阻塞
- **exchange 是双向的**，双方同时交换
- **适用场景较少**，大部分场景可以用其他并发工具替代

## 易错点与踩坑

### 1. 中断会导致数据丢失或永久阻塞

```java
Exchanger<String> exchanger = new Exchanger<>();

Thread t1 = new Thread(() -> {
    try {
        String data = "t1-data";
        String received = exchanger.exchange(data);  // 等待配对
        System.out.println("t1 收到: " + received);
    } catch (InterruptedException e) {
        // ❌ 如果被中断，t1 的数据还没交换出去
        System.out.println("t1 被中断，数据丢失！");
        Thread.currentThread().interrupt();
    }
});

Thread t2 = new Thread(() -> {
    try {
        Thread.sleep(5000);  // 模拟延迟
        String data = "t2-data";
        String received = exchanger.exchange(data);
        System.out.println("t2 收到: " + received);
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
});

t1.start();
t2.start();
Thread.sleep(1000);
t1.interrupt();  // t1 在等待时被中断

// ⚠️ t2 会一直等待配对，永久阻塞
```

### 2. 空槽和花粉问题的性能影响

```java
Exchanger<Node> exchanger = new Exchanger<>();

// ❌ Exchanger 内部使用槽（slot）来暂存数据
// 单槽设计：同一时刻只能有一方放数据

// 场景：高频交换
for (int i = 0; i < 10000; i++) {
    exchanger.exchange(data);  // 每次都要竞争唯一槽位
}

// ⚠️ 性能问题：
// 1. 只有一个槽，多线程竞争激烈
// 2. CAS 失败率高
// 3. 缓存一致性：多个 CPU 竞争同一个缓存行

// ✅ 解决：JDK9+ 可以用 StripedExchanger（但没有公开 API）
// 或者使用其他并发队列替代
```

### 3. 只能两个线程配对，奇数线程会出问题

```java
Exchanger<String> exchanger = new Exchanger<>();

// ❌ 三个线程调用 exchange
// t1 和 t2 配对成功，t3 永久阻塞
exchanger.exchange("data1");  // t1
exchanger.exchange("data2");  // t2
exchanger.exchange("data3");  // t3，永远等待

// ✅ 正确做法：确保偶数个线程
// 或者使用 SynchronousQueue（容量为 0 的阻塞队列）
SynchronousQueue<String> queue = new SynchronousQueue<>();

// ✅ 或者用 ConcurrentLinkedQueue 自己实现配对
// （但需要额外的同步逻辑）
```

### 4. 内存可见性问题

```java
Exchanger<String> exchanger = new Exchanger<>();

// ❌ 如果不用 volatile 或 final，交换的数据可能不可见
Object data = new Object();
Object received = exchanger.exchange(data);

// ⚠️ 虽然 Exchanger 内部有内存屏障
// 但如果数据对象内部状态被修改，不一定能保证可见性

// ✅ 正确做法：数据对象应该是不可变的，或者用 final
final String data = "t1-data";
final String received = exchanger.exchange(data);

// ✅ 或者数据对象本身是线程安全的
AtomicReference<String> atomicData = new AtomicReference<>("t1-data");
AtomicReference<String> receivedData = new AtomicReference<>();
exchanger.exchange(atomicData.get());
```

## 关联知识点

