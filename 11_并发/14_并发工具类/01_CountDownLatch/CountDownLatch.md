
# CountDownLatch

## 核心结论

CountDownLatch 是一个**一次性**同步辅助类，允许一个或多个线程等待其他线程完成操作。内部基于 **AQS 共享模式**实现，通过 `countDown()` 递减计数器，计数到 0 时唤醒所有等待线程。

## 深度解析

### 核心机制

```
初始化 count = N
线程调用 await() → 如果 count > 0，进入 AQS 等待队列
线程调用 countDown() → count 递减，count = 0 时唤醒所有等待线程
```

### 关键方法

| 方法 | 说明 |
|------|------|
| `CountDownLatch(int count)` | 初始化计数器 |
| `await()` | 当前线程阻塞，直到 count = 0 |
| `await(timeout, unit)` | 带超时的等待 |
| `countDown()` | 计数器减 1 |
| `getCount()` | 获取当前计数 |

### AQS 实现原理

- **内部维护 Sync 类**（继承 AQS），`state` 表示剩余计数
- `await()` → 调用 `tryAcquireShared(1)`，判断 `state == 0` 则返回 1（成功），否则进入共享等待队列
- `countDown()` → 调用 `tryReleaseShared(1)`，CAS 递减 state，成功后 `doReleaseShared()` 唤醒等待线程
- **不可重置**：count 减到 0 后无法重用

## 代码示例

### 基本用法

```java
CountDownLatch latch = new CountDownLatch(3);

for (int i = 0; i < 3; i++) {
    new Thread(() -> {
        try {
            TimeUnit.SECONDS.sleep(1);
            System.out.println(Thread.currentThread().getName() + " 完成");
        } finally {
            latch.countDown();
        }
    }, "Worker-" + i).start();
}

latch.await(); // 主线程等待
System.out.println("所有任务完成");
```

### 主线程等待多个子任务

```java
ExecutorService pool = Executors.newFixedThreadPool(10);
CountDownLatch latch = new CountDownLatch(taskList.size());

for (Task task : taskList) {
    pool.submit(() -> {
        try {
            task.execute();
        } finally {
            latch.countDown();
        }
    });
}

latch.await(30, TimeUnit.SECONDS); // 最多等30秒
pool.shutdown();
```

## 注意事项

- **countDown() 必须在 finally 中调用**，确保异常时也能递减
- **count 不可重置**，需要重用场景用 CyclicBarrier
- **countDown() 放在 finally 而非 catch**，因为正常执行也需要减计数
- **count 初始化后不能修改**，设错就不可恢复

# CountDownLatch使用场景

## 核心结论

CountDownLatch 典型场景：**多线程并行完成后汇总结果**、**主线程等待初始化完成**、**并发测试模拟**。

## 深度解析

### 场景一：多线程并行处理汇总

```java
// 并行查询多个数据源，全部完成后合并
CountDownLatch latch = new CountDownLatch(3);
Map<String, Object> result = new ConcurrentHashMap<>();

new Thread(() -> {
    try { result.put("user", userService.query(id)); }
    finally { latch.countDown(); }
}).start();

new Thread(() -> {
    try { result.put("order", orderService.query(id)); }
    finally { latch.countDown(); }
}).start();

new Thread(() -> {
    try { result.put("asset", assetService.query(id)); }
    finally { latch.countDown(); }
}).start();

latch.await();
// result 中三个数据都就绪
```

### 场景二：等待初始化完成

```java
// 应用启动时等待多个组件初始化
CountDownLatch initLatch = new CountDownLatch(3);

// 配置中心初始化
new Thread(() -> {
    configCenter.init();
    initLatch.countDown();
}).start();

// 数据库连接池初始化
new Thread(() -> {
    dataSource.init();
    initLatch.countDown();
}).start();

// 缓存预热
new Thread(() -> {
    cache.warmUp();
    initLatch.countDown();
}).start();

initLatch.await();
log.info("所有组件初始化完成，开始接收请求");
server.start();
```

### 场景三：并发测试

```java
// 模拟 1000 个线程同时发起请求
int threadCount = 1000;
CountDownLatch startLatch = new CountDownLatch(1);  // 发令枪
CountDownLatch doneLatch = new CountDownLatch(threadCount);  // 完成计数

for (int i = 0; i < threadCount; i++) {
    new Thread(() -> {
        try {
            startLatch.await(); // 所有线程等待发令
            apiClient.call();   // 同时执行
        } catch (Exception e) {
            // 记录异常
        } finally {
            doneLatch.countDown();
        }
    }).start();
}

long start = System.currentTimeMillis();
startLatch.countDown(); // 发令：所有线程同时开始
doneLatch.await(60, TimeUnit.SECONDS);
long cost = System.currentTimeMillis() - start;
log.info("1000并发请求耗时: {}ms", cost);
```

### 场景四：任务分片处理

```java
// 大任务拆分为 N 个子任务，并行执行后汇总
List<List<Data>> shards = splitData(dataList, 10);
CountDownLatch latch = new CountDownLatch(shards.size());
List<ProcessResult> results = new ArrayList<>();

for (List<Data> shard : shards) {
    executor.submit(() -> {
        try {
            ProcessResult r = process(shard);
            synchronized (results) { results.add(r); }
        } finally {
            latch.countDown();
        }
    });
}

latch.await();
Result finalResult = merge(results);
```

## 易错点与踩坑

### 1. await() 可被中断导致任务失败

```java
CountDownLatch latch = new CountDownLatch(3);

// ❌ 线程被中断时，await() 抛 InterruptedException
try {
    latch.await();  // 如果被中断，直接抛异常，不会重试
} catch (InterruptedException e) {
    // ⚠️ 中断状态被清除，latch 仍然是 3
    // 主线程可能认为任务失败
    Thread.currentThread().interrupt();  // 必须恢复中断状态
    throw new RuntimeException("任务被中断", e);
}

// ✅ 正确做法：使用带超时的 await
boolean success = latch.await(30, TimeUnit.SECONDS);
if (!success) {
    // 超时处理
}
```

### 2. countDown() 调用次数多于初始化值

```java
// ❌ 多次调用 countDown() 不会出错，但不会重复唤醒
CountDownLatch latch = new CountDownLatch(1);

// 第一次：count 变为 0，唤醒等待线程
latch.countDown();
// 第二次及以后：count 已经是 0，不会再变
latch.countDown();  // 无效果
latch.countDown();  // 无效果

// ⚠️ 常见错误：在线程池场景中，任务复用导致多调用
ExecutorService pool = Executors.newFixedThreadPool(3);
for (int i = 0; i < 3; i++) {
    pool.submit(() -> {
        doWork();
        latch.countDown();  // 正确：每个任务只调用一次
    });
}

// ❌ 错误写法：submit 了 10 个任务，但 latch 只设置了 3
latch.countDown();  // 在 submit 里
latch.countDown();  // 在 submit 里
latch.countDown();  // 在 submit 里
// 其他 7 个任务也会调用 countDown()，但已经无效果
```

### 3. 子线程异常不会影响 latch 状态

```java
// ❌ 某个子线程抛异常，latch 仍然会减到 0
CountDownLatch latch = new CountDownLatch(3);

Thread t1 = new Thread(() -> { latch.countDown(); });        // 正常
Thread t2 = new Thread(() -> { throw new RuntimeException(); });  // 异常！
Thread t3 = new Thread(() -> { latch.countDown(); });        // 正常

t1.start(); t2.start(); t3.start();
latch.await();  // 仍然会通过，因为 latch 确实减到 0 了

// ✅ 正确做法：捕获子线程异常，或用其他同步机制
Thread t2 = new Thread(() -> {
    try {
        doSomething();
    } finally {
        latch.countDown();
    }
});
```

### 4. 不可重置，不能用于循环场景

```java
// ❌ CountDownLatch 是一次性的，不能重用
CountDownLatch latch = new CountDownLatch(3);

// 第一轮
latch.await();  // 等待 count 到 0
// 第二轮：latch 已经到 0，不会再等待
latch.await();  // 立即返回，不是等待新一轮！

// ✅ 如果需要可重用的同步，用 CyclicBarrier
CyclicBarrier barrier = new CyclicBarrier(3);
barrier.await();  // 第一轮
barrier.reset();  // 可重置
barrier.await();  // 第二轮
```

### 5. await() 在 count=0 后调用会立即返回

```java
CountDownLatch latch = new CountDownLatch(1);
latch.countDown();

// ✅ 这个行为是正确的
latch.await();  // 立即返回，不会阻塞

// ⚠️ 但如果业务逻辑依赖"等待完成事件"，可能在 countDown 之前就调用了 await
// 这时应该确保 countDown 在 await 之前被调用
```

## 关联知识点
