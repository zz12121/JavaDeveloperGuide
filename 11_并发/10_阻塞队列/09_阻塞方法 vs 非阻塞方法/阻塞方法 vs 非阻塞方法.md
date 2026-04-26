---
title: 阻塞方法 vs 非阻塞方法
tags:
  - Java/并发
  - 对比型
module: 10_阻塞队列
created: 2026-04-18
---

# 阻塞方法 vs 非阻塞方法

## 核心结论

BlockingQueue 提供四组不同行为的 API：**抛异常 / 返回值 / 阻塞 / 超时**。根据场景选择合适的方法：实时响应用阻塞，避免死锁用超时。

## 深度解析

### 四组 API 对比

| 类型 | 插入 | 移除 | 检查 | 说明 |
|------|------|------|------|------|
| **抛异常** | add(e) | remove() | element() | 队列满/空时抛异常 |
| **返回值** | offer(e) | poll() | peek() | 满返回false，空返回null |
| **阻塞** | put(e) | take() | — | 满阻塞，空阻塞 |
| **超时** | offer(e,time,unit) | poll(time,unit) | — | 超时返回false/null |

### 详细说明

#### 抛异常
```java
queue.add(e);      // 满时抛 IllegalStateException
queue.remove();    // 空时抛 NoSuchElementException
queue.element();   // 空时抛 NoSuchElementException
```

#### 返回值（推荐日常使用）
```java
boolean success = queue.offer(e); // 满返回 false
E item = queue.poll();            // 空返回 null
E item = queue.peek();            // 空返回 null，不删除
```

#### 阻塞（生产者-消费者模式）
```java
queue.put(e);       // 满时阻塞直到有空间
E item = queue.take(); // 空时阻塞直到有元素
```

#### 超时（避免无限等待）
```java
boolean success = queue.offer(e, 3, TimeUnit.SECONDS); // 3秒超时
E item = queue.poll(3, TimeUnit.SECONDS); // 3秒超时
```

### 场景选择

| 场景 | 推荐方法 | 原因 |
|------|---------|------|
| 生产者-消费者 | put/take | 阻塞等待，自动平衡 |
| 定时任务 | offer/poll(timeout) | 超时返回，避免积压 |
| 尝试性操作 | offer/poll | 非阻塞，立即返回 |
| 避免死锁 | tryLock + offer/poll | 超时防止永久阻塞 |
| 批量处理 | drainTo | 批量移除，高效 |

## 易错点/踩坑

1. **add() 的隐藏陷阱**：add() 是从 Collection 继承的方法，队列满时抛 `IllegalStateException` 而非返回 false。如果用 add() 做任务入队且不 catch 异常，任务会丢失且线程可能中断。建议用 offer() 或 put()。

2. **take() 无超时版本会永久阻塞**：如果消费者线程调用了 take() 且队列永远为空（如生产者已经退出），消费者将永久阻塞。推荐用 `poll(timeout)` 并处理返回 null 的情况。

3. **element() 和 peek() 的区别**：element() 空时抛异常，peek() 空时返回 null。面试中经常考察这个区别。

4. **drainTo() 不等待**：drainTo() 是非阻塞的，有多少取多少。如果队列为空，返回空列表，不会等待。适合批量消费但不保证数量。

5. **混合使用注意锁一致性**：不要在同一线程中交替使用 put/take 和 offer/poll 做同一队列的操作——put/take 的阻塞语义可能与其他方法的非阻塞语义产生竞争，增加死锁风险。

6. **interrupt 中断阻塞操作**：put/take 和超时版本都响应中断。线程被 interrupt 时，这些方法抛出 InterruptedException，不会继续阻塞。正确做法是 catch 后恢复中断标志。

## 关联知识点
