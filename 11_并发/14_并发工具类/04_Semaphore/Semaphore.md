
# Semaphore

## 核心结论

Semaphore（信号量）用于**控制同时访问共享资源的线程数量**。内部基于 AQS 共享模式实现，通过 `acquire()` 获取许可、`release()` 释放许可，许可数不足时线程阻塞等待。

## 深度解析

### 核心机制

```
初始化 permits = N

Thread-1: acquire() → permits 变为 N-1，继续执行
Thread-2: acquire() → permits 变为 N-2，继续执行
...
Thread-N: acquire() → permits 变为 0，继续执行
Thread-N+1: acquire() → permits 为 0，阻塞等待

Thread-1: release() → permits 变为 1，唤醒 Thread-N+1
```

### 关键方法

| 方法 | 说明 |
|------|------|
| `Semaphore(int permits)` | 创建指定许可数 |
| `Semaphore(int permits, boolean fair)` | 公平/非公平模式 |
| `acquire()` | 获取 1 个许可，不足则阻塞 |
| `acquire(n)` | 获取 n 个许可 |
| `release()` | 释放 1 个许可 |
| `release(n)` | 释放 n 个许可 |
| `tryAcquire()` | 非阻塞尝试获取 |
| `tryAcquire(timeout, unit)` | 带超时尝试获取 |
| `availablePermits()` | 可用许可数 |
| `drainPermits()` | 获取所有可用许可 |

### AQS 实现原理

- **Sync 继承 AQS**，`state` 表示许可数
- `acquire()` → `tryAcquireShared(1)`，CAS 递减 state，失败则进入等待队列
- `release()` → `tryReleaseShared(1)`，CAS 递增 state，成功后唤醒等待线程
- **公平模式**：`tryAcquireShared` 中先检查 `hasQueuedPredecessors()`，有排队线程则不插队
- **非公平模式**（默认）：直接 CAS 尝试获取，失败才排队

## 代码示例

### 限流控制

```java
// 最多同时 5 个线程访问数据库
Semaphore semaphore = new Semaphore(5);

public void queryDatabase(String sql) {
    try {
        semaphore.acquire();
        doQuery(sql); // 最多5个线程同时执行
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    } finally {
        semaphore.release();
    }
}
```

### 资源池

```java
// 模拟连接池，最多10个连接
Semaphore connections = new Semaphore(10);

public Connection getConnection() throws InterruptedException {
    connections.acquire();
    return createConnection();
}

public void returnConnection(Connection conn) {
    closeConnection(conn);
    connections.release();
}
```

### Semaphore(1) 当互斥锁

```java
Semaphore mutex = new Semaphore(1);
mutex.acquire();
try {
    // 临界区
} finally {
    mutex.release();
}
```

## 注意事项

- **release() 必须在 finally 中**，防止许可泄漏
- **acquire 和 release 不要求在同一个线程**，但通常应配对
- **Semaphore(1) 相当于互斥锁**，但 acquire/release 可以跨线程
- **公平模式性能较低**，非公平模式吞吐更高


# Semaphore使用场景

## 核心结论

Semaphore 最典型场景是**限流**——控制同时访问某个资源的并发线程数量，如数据库连接池、接口限流、文件句柄管理等。

## 深度解析

### 场景一：接口限流

```java
// 限制同一接口最多 100 个并发请求
Semaphore rateLimit = new Semaphore(100);

@GetMapping("/api/data")
public Result getData() {
    if (!rateLimit.tryAcquire()) {
        return Result.fail("系统繁忙，请稍后重试");
    }
    try {
        return service.query();
    } finally {
        rateLimit.release();
    }
}
```

### 场景二：数据库连接池模拟

```java
// 简易连接池，最多 20 个连接
public class SimpleConnectionPool {
    private final Semaphore semaphore = new Semaphore(20);
    private final BlockingQueue<Connection> pool = new LinkedBlockingQueue<>(20);

    public Connection getConnection() throws InterruptedException {
        semaphore.acquire();
        Connection conn = pool.poll();
        if (conn == null) {
            conn = DriverManager.getConnection(url);
        }
        return conn;
    }

    public void releaseConnection(Connection conn) {
        if (pool.offer(conn)) {
            semaphore.release();
        } else {
            close(conn);
            semaphore.release();
        }
    }
}
```

### 场景三：文件读写限流

```java
// 限制同时打开的文件数
Semaphore fileLimit = new Semaphore(50);

public void processFile(String path) {
    try {
        fileLimit.acquire();
        try (InputStream is = new FileInputStream(path)) {
            // 读取处理
        }
    } catch (Exception e) {
        log.error("处理失败", e);
    } finally {
        fileLimit.release();
    }
}
```

### 场景四：停车场的经典比喻

```
停车场有 10 个车位（Semaphore(10)）
车辆进入 → acquire()，有车位则进入，无车位则等待
车辆离开 → release()，释放车位，等待的车辆进入
```

### 场景对比

|场景|permits|说明|
|---|---|---|
|数据库连接池|最大连接数|控制连接数|
|接口限流|最大并发数|超限快速失败|
|文件操作|最大文件句柄数|防止 Too many open files|
|线程池任务限流|最大任务数|防止 OOM|
|缓存预热|并行度|控制预热的并发数|

## 易错点与踩坑

### 1. acquire/release 必须配对，否则资源泄漏

```java
Semaphore semaphore = new Semaphore(3);

// ❌ acquire() 后如果抛异常，permit 永远不会释放
try {
    semaphore.acquire();  // 获取 permit
    doWork();  // 如果这里抛异常
    // ⚠️ permit 永远不会 release！
} catch (InterruptedException e) {
    e.printStackTrace();
}

// ✅ 正确做法：try-finally 确保释放
try {
    semaphore.acquire();
    try {
        doWork();  // 可能抛异常
    } finally {
        semaphore.release();  // 无论如何都会释放
    }
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}

// ✅ JDK18+ 推荐：tryAcquire（不推荐，会导致资源泄漏）
// semaphore.acquire() 不会抛出 InterruptedException 的重载版本
// 使用 acquireUninterruptibly() 也要确保在 finally 中释放
```

### 2. acquire 和 release 不能用反

```java
Semaphore semaphore = new Semaphore(1);

// ❌ release 多于 acquire → permit 超出初始值
semaphore.release();  // permit = 2
semaphore.release();  // permit = 3
// 后续所有 acquire 都会成功，限流失效

// ❌ acquire 多于 release → 其他线程永久阻塞
semaphore.acquire();  // permit = 0
semaphore.acquire();  // 阻塞
// ⚠️ 如果没有其他地方 release，这个线程会永久阻塞

// ✅ 正确模式：acquire 和 release 必须平衡
semaphore.acquire();  // permit = N-1
// ... do work ...
semaphore.release();  // permit = N
```

### 3. fair vs nonfair 模式的选择

```java
// ❌ 默认是非公平模式，可能导致线程饥饿
Semaphore semaphore = new Semaphore(3, false);  // nonfair

// 场景：高频写线程 + 低频读线程
// 非公平：写线程可能一直抢在读线程前面
// 导致读线程长时间得不到 permit

// ✅ 如果需要公平访问，用公平模式
Semaphore fairSemaphore = new Semaphore(3, true);  // fair

// ⚠️ 公平模式性能更低
// 原因：每次获取都要检查队列前面的线程是否等待更久
// CAS 失败率更高

// ✅ 正确选择：
// - 限流/资源池：非公平（性能优先）
// - 公平调度/防止饥饿：公平（正确性优先）
```

### 4. tryAcquire 的陷阱

```java
Semaphore semaphore = new Semaphore(1);

// ❌ 误以为 tryAcquire 会阻塞等待
boolean acquired = semaphore.tryAcquire();  // 非阻塞，立即返回

// ✅ 带超时的 tryAcquire
boolean acquired = semaphore.tryAcquire(5, TimeUnit.SECONDS);  // 等待 5 秒

// ⚠️ 常见错误：在循环中使用 tryAcquire 而不处理失败
while (!semaphore.tryAcquire()) {
    Thread.sleep(100);  // ❌ 空转浪费 CPU
}

// ✅ 正确做法：
if (!semaphore.tryAcquire(5, TimeUnit.SECONDS)) {
    // 超时处理
}

// ✅ 或者使用 acquire()（会阻塞）
semaphore.acquire();  // 阻塞直到获取成功
```

## 关联知识点
