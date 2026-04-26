---
title: CAS原理
tags:
  - Java/并发
  - 原理型
module: 05_CAS
created: 2026-04-18
---

# CAS原理（三个操作数：内存位置V/期望值A/新值B）

## 先说结论

CAS（Compare And Swap）是一种无锁并发原语，通过三个操作数——内存位置V、期望值A、新值B——实现原子更新：当且仅当V的值等于A时，才将V更新为B，否则重试或返回失败。它是整个 JUC 并发框架的基石。

## 深度解析

### 什么是 CAS

CAS 是 CPU 提供的原子指令，Java 通过 `Unsafe` 类封装了该操作。其核心逻辑：

```
boolean CAS(V, A, B) {
    if (V == A) {
        V = B;
        return true;  // 成功
    }
    return false;     // 失败，V已被其他线程修改
}
```

### 三个操作数

| 操作数 | 含义 | 示例 |
|--------|------|------|
| **V（Value）** | 内存中的实际值 | 主内存中变量的当前值 |
| **A（Expected）** | 预期旧值 | 线程上次读取到的值 |
| **B（New）** | 要写入的新值 | 线程计算后的新值 |

### CAS 执行流程

```
线程1                              主内存(V=10)
  │                                  │
  ├── 读取 V=10 (A=10)              │
  ├── 计算 B=11                      │
  │                                  │
  │        线程2: CAS(V,10,11) ✓     │ V→11
  │                                  │
  ├── CAS(V, A=10, B=11)            │
  ├── V=11 ≠ A=10 → 失败！           │
  ├── 重新读取 V=11 (A=11)          │
  ├── 重新计算 B=12                  │
  ├── CAS(V, A=11, B=12)            │
  ├── V=11 == A=11 → 成功！         │ V→12
  │                                  │
```

### CAS 与 volatile 的配合

CAS 本身只保证"比较+交换"这一步的原子性，但读取 V 时的可见性需要 `volatile` 保证。JUC 中所有 CAS 操作的字段都用 `volatile` 修饰。

## 易错点/踩坑

- ❌ 认为 CAS 完全无锁——CAS 底层仍然需要 CPU 的锁总线或缓存锁
- ❌ 认为 CAS 能保证复合操作的原子性——CAS 只保证单个变量的原子更新
- ✅ CAS 失败时通常配合自旋（循环重试）使用
- ✅ 竞争激烈时 CAS 自旋会浪费 CPU 资源

## 代码示例

```java
// 模拟 CAS 原理
public class SimpleCAS {
    private volatile int value;

    // CAS 操作（实际由 Unsafe.compareAndSwapInt 实现）
    public boolean compareAndSet(int expected, int newValue) {
        synchronized (this) {
            if (value == expected) {
                value = newValue;
                return true;
            }
            return false;
        }
    }

    // 自旋 CAS 实现线程安全递增
    public int increment() {
        int old;
        int newValue;
        do {
            old = value;           // 1. 读当前值
            newValue = old + 1;    // 2. 计算新值
        } while (!compareAndSet(old, newValue)); // 3. CAS 更新
        return newValue;
    }
}
```

## 关联知识点
