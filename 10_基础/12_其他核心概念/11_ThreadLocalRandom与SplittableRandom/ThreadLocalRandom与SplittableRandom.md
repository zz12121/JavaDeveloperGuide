---
title: ThreadLocalRandom与SplittableRandom
tags:
  - Java/并发
  - 随机数
  - 高性能
module: 12_其他核心概念
created: 2026-04-25
---

# ThreadLocalRandom与SplittableRandom

## Random 的局限性

`java.util.Random` 是 JDK 1.0 就存在的随机数生成器，但存在严重的**线程安全问题**：

```java
// Random 的 nextInt() 实现
private final AtomicLong seed;
protected int next(int bits) {
    long oldseed, nextseed;
    AtomicLong seed = this.seed;
    do {
        oldseed = seed.get();
        nextseed = (oldseed * multiplier + addend) & mask;
    } while (!seed.compareAndSet(oldseed, nextseed));  // CAS 竞争！
    return (int)(nextseed >>> (48 - bits));
}
```

问题在于 `compareAndSet()` 在高并发下会产生**大量线程竞争**：
- 每个线程调用 `nextInt()` 都需要竞争同一个 `AtomicLong`
- 多线程场景下，性能急剧下降

## ThreadLocalRandom

### 核心原理

JDK 7 引入的 `ThreadLocalRandom` 为**每个线程维护独立的 Random 实例**，彻底消除了线程竞争：

```java
// ThreadLocalRandom 内部实现（简化）
class ThreadLocalRandom {
    // 每个线程独立的种子
    long threadLocalRandomSeed;

    // 获取当前线程的随机数生成器
    public static ThreadLocalRandom current() {
        Thread currentThread = Thread.currentThread();
        ThreadLocalRandom rand = threadLocalRandomHolder.get(currentThread);
        return rand;
    }

    public int nextInt(int bound) {
        long m = threadLocalRandomSeed;
        long r = (m ^ (m >>> 33)) * 0xff51afd7ed558ccdL;  // 无锁计算
        threadLocalRandomSeed = r;
        return (int)(r % bound);
    }
}
```

### 基本用法

```java
import java.util.concurrent.ThreadLocalRandom;

// 获取当前线程的随机数生成器
ThreadLocalRandom random = ThreadLocalRandom.current();

// 生成随机整数 [0, bound)
int id = ThreadLocalRandom.current().nextInt(100);      // 0~99

// 生成指定范围 [origin, bound)
int score = ThreadLocalRandom.current().nextInt(60, 100); // 60~99

// 生成 long
long value = ThreadLocalRandom.current().nextLong(1000L);

// 生成 double
double rate = ThreadLocalRandom.current().nextDouble(0.0, 1.0);

// 生成 boolean
boolean flag = ThreadLocalRandom.current().nextBoolean();

// 生成随机数组
int[] arr = ThreadLocalRandom.current().ints(100, 0, 100)
    .toArray();  // 100 个 0~99 的随机整数
```

### 常见错误

```java
// 错误 1：在主线程创建，子线程无法继承
Random r = new Random();  // ❌ 多线程竞争
ThreadLocalRandom tlr = ThreadLocalRandom.current();  // ✅ 在需要的线程中获取

// 错误 2：fork/join 场景下种子共享问题
// ThreadLocalRandom 的种子是线程本地的，
// 但在 fork/join 的工作线程中，子线程应该有自己的种子
ForkJoinPool pool = ForkJoinPool.commonPool();
pool.submit(() -> {
    // 在这个线程中获取
    ThreadLocalRandom.current().nextInt(100);
});

// 错误 3：在新线程中使用前未调用 current()
Thread t = new Thread(() -> {
    ThreadLocalRandom r = ThreadLocalRandom.current();  // ✅ 必须调用
    r.nextInt(100);
});
```

### Random vs ThreadLocalRandom 性能对比

```java
public class RandomBenchmark {
    public static void main(String[] args) {
        int iterations = 10_000_000;
        int threads = 8;

        // Random：大量 CAS 竞争
        long rStart = System.nanoTime();
        Random random = new Random();
        IntStream.range(0, iterations)
            .parallel()
            .forEach(i -> random.nextInt());
        long rTime = System.nanoTime() - rStart;

        // ThreadLocalRandom：无竞争
        long tlrStart = System.nanoTime();
        IntStream.range(0, iterations)
            .parallel()
            .forEach(i -> ThreadLocalRandom.current().nextInt());
        long tlrTime = System.nanoTime() - tlrStart;

        System.out.println("Random: " + rTime / 1_000_000 + "ms");
        System.out.println("ThreadLocalRandom: " + tlrTime / 1_000_000 + "ms");
    }
}
```

## SplittableRandom

### 适用场景

`SplittableRandom` 专为 **fork/join 框架和并行流** 设计，支持 `split()` 操作，每个 split 生成一个新的独立随机数生成器：

```java
import java.util.SplittableRandom;
```

### split() 原理

```
初始 SplittableRandom（主线程）
    ├── split() → 子生成器 1（独立种子）
    │   ├── split() → 孙生成器 1a（独立种子）
    │   └── split() → 孙生成器 1b（独立种子）
    └── split() → 子生成器 2（独立种子）
```

每次 `split()` 生成一个新实例，**种子完全不同**，保证并行任务之间的随机性独立。

### 基本用法

```java
// 创建主生成器（通常在主线程）
SplittableRandom mainRandom = new SplittableRandom();

// 分裂：并行任务各持有一个独立的生成器
SplittableRandom task1 = mainRandom.split();
SplittableRandom task2 = mainRandom.split();
SplittableRandom task3 = mainRandom.split();

// 每个生成器独立使用
int rand1 = task1.nextInt(100);
int rand2 = task2.nextInt(100);

// 生成流（并行友好）
IntStream ints = mainRandom.ints(1000, 0, 100);
LongStream longs = mainRandom.longs(1000, 0, Long.MAX_VALUE);
DoubleStream doubles = mainRandom.doubles(1000);
```

### 与并行流配合

```java
// 错误的并行流随机数用法
Stream.generate(() -> new Random().nextInt())  // ❌ 创建大量 Random 实例
    .limit(1_000_000)
    .parallel()
    .count();

// 正确用法：每个并行任务使用 SplittableRandom.split()
List<Integer> randomList = IntStream.range(0, 1_000_000)
    .parallel()
    .mapToObj(i -> {
        SplittableRandom sr = new SplittableRandom();  // 每次创建新的
        return sr.nextInt(100);
    })
    .collect(Collectors.toList());

// 更优：预分裂后分配给各并行任务
SplittableRandom sr = new SplittableRandom();
List<SplittableRandom> randoms = IntStream.range(0, 8)
    .mapToObj(i -> sr.split())
    .collect(Collectors.toList());
```

### 性能对比

| 随机数生成器 | 线程安全 | fork/join 友好 | 适用场景 |
|------------|---------|---------------|---------|
| `Random` | ⚠️ CAS 竞争 | ❌ | 单线程、简单场景 |
| `ThreadLocalRandom` | ✅ 无竞争 | ❌ | 多线程、非 fork/join |
| `SplittableRandom` | ✅ 独立种子 | ✅ | fork/join、并行流 |

## JDK 17+ SecureRandom

### 使用场景

`SecureRandom` 用于**安全敏感**场景（密码学、令牌生成）：

```java
import java.security.SecureRandom;

// 生成安全随机字节
SecureRandom secure = new SecureRandom();
byte[] bytes = new byte[32];
secure.nextBytes(bytes);

// 生成安全随机 UUID
String token = new BigInteger(130, secure).toString(32);

// 生成安全随机字符串（密码、令牌）
public static String generateSecureToken(int length) {
    SecureRandom random = new SecureRandom();
    String chars = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789";
    StringBuilder sb = new StringBuilder(length);
    for (int i = 0; i < length; i++) {
        sb.append(chars.charAt(random.nextInt(chars.length())));
    }
    return sb.toString();
}
```

## 总结

| 方法 | 特性 | 使用建议 |
|------|------|----------|
| `Random` | CAS 竞争，性能低 | ❌ 不推荐多线程使用 |
| `ThreadLocalRandom` | 线程本地，无竞争 | ✅ 多线程首选 |
| `SplittableRandom` | 支持 split，独立种子 | ✅ fork/join 首选 |
| `SecureRandom` | 密码学安全，慢 | ✅ 安全场景专用 |

## 面试关联

### Q1：ThreadLocalRandom 为什么比 Random 快？

**A：** Random 每个方法调用都执行 `compareAndSet()` CAS 操作来更新种子，高并发下大量线程竞争同一个 `AtomicLong`。ThreadLocalRandom 为每个线程维护独立的种子，完全无竞争，直接在本地计算。

### Q2：ThreadLocalRandom 的子线程会继承父线程的种子吗？

**A：** **不会**。ThreadLocalRandom 的种子是线程本地的（类似 ThreadLocal），子线程调用 `current()` 时会获取或初始化自己的种子，与父线程无关。这正是与 ThreadLocal 的关键区别——ThreadLocal 允许值传递，而 ThreadLocalRandom 每个线程都是独立的。

### Q3：SplittableRandom 的 split 操作有什么用？

**A：** 在 fork/join 场景中，每个子任务需要独立的随机数生成器以保证随机性。`split()` 生成的新生成器拥有完全独立的种子，性能比创建新 `SplittableRandom` 实例更好（内部有优化）。

### Q4：生产环境随机数选择建议？

**A：**
- 普通业务随机数：`ThreadLocalRandom.current()`
- 并行流 + fork/join：`SplittableRandom` + split
- 密码学/安全令牌：`SecureRandom`
- 避免使用：`new Random()`（多线程性能差）
