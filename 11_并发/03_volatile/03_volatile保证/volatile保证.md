---
title: volatile保证
tags:
  - Java/并发
  - 原理型
module: 03_volatile
created: 2026-04-18
---

# volatile保证（禁止指令重排序，保证可见性）

## 先说结论

volatile 通过**内存屏障（Memory Barrier）**实现两大保证：**禁止指令重排序**（有序性）和**保证可见性**（缓存一致性）。JMM 针对编译器和处理器分别制定了 volatile 重排序规则表，确保 volatile 变量的读写操作不会被错误地重排。

## 深度解析

### 有序性保证：volatile 重排序规则

JMM 规定了以下规则（第二操作为 volatile 时）：

| 第一个操作 | 第二个操作 | 是否允许重排序 |
|-----------|-----------|--------------|
| 普通读/写 | volatile 写 | ❌ 不允许 |
| 普通读/写 | volatile 读 | ❌ 不允许 |
| volatile 写 | 普通读/写 | ❌ 不允许 |
| volatile 写 | volatile 读 | ❌ 不允许 |
| volatile 读 | 普通读/写 | ⚠️ 允许（但实际很少发生） |

**核心**：volatile 写前面的操作不会被重排到写之后，volatile 读后面的操作不会被重排到读之前。

### 可见性保证：缓存一致性

- volatile 写之后，JMM 会将工作内存中的所有共享变量刷新到主内存
- volatile 读之前，JMM 会使工作内存中的所有共享变量失效，强制从主内存重新加载
- 这种保证不仅针对 volatile 变量本身，还影响 volatile 操作前后附近的共享变量

### happens-before 关系

volatile 写 → 后续对同一变量的 volatile 读（volatile HB 规则）

## 易错点/踩坑

- ❌ 认为 volatile 只保证被修饰变量的可见性
- ✅ volatile 操作会同步工作内存中的所有共享变量
- ❌ 以为 volatile 读后面的操作一定不会被重排到读之前
- ✅ volatile 读后面的普通写操作理论上可以被重排（但 JIT 通常不会这么做）

## 代码示例

```java
// volatile 有序性：DCL 双重检查锁单例
public class Singleton {
    // volatile 禁止对象初始化指令的重排序
    // 避免 B 线程拿到未完全初始化的对象
    private static volatile Singleton instance;

    public static Singleton getInstance() {
        if (instance == null) {                    // 第一次检查
            synchronized (Singleton.class) {
                if (instance == null) {            // 第二次检查
                    instance = new Singleton();     // 可能重排序！
                }
            }
        }
        return instance;
    }
}

// new Singleton() 字节码层面分三步：
// 1. memory = allocate()        // 分配内存
// 2. ctorInstance(memory)        // 初始化对象
// 3. instance = memory           // 引用指向内存
// 没有 volatile 时，步骤 2 和 3 可能被重排序为 1→3→2
// 导致 B 线程判断 instance != null 但对象尚未初始化完成
```

```java
// volatile 可见性：配置更新对所有线程立即可见
public class ConfigManager {
    private volatile boolean updated = false;
    private volatile String config = "default";

    // 写线程
    public void updateConfig(String newConfig) {
        this.config = newConfig;    // 普通写
        this.updated = true;        // volatile 写，前面的 config 写也会被刷新
    }

    // 读线程
    public String getConfig() {
        if (updated) {              // volatile 读，触发所有共享变量重新加载
            return config;          // 能读到最新值
        }
        return config;
    }
}
```

## 图解/流程

```
DCL 单例中 volatile 禁止重排序的作用：

无 volatile 时可能的重排序：
  线程A                           线程B
  ┌──────────────────┐          ┌──────────────────┐
  │ 1. 分配内存空间   │          │                  │
  │ 3. 引用指向内存   │ ──可见──>│ 判断!=null       │
  │ 2. 初始化对象     │          │ 使用未初始化对象  │ ← 危险！
  └──────────────────┘          └──────────────────┘

有 volatile 时：
  线程A                           线程B
  ┌──────────────────┐          ┌──────────────────┐
  │ 1. 分配内存空间   │          │                  │
  │ 2. 初始化对象     │          │ 判断==null       │
  │ StoreStore屏障    │          │                  │
  │ 3. 引用指向内存   │ ──可见──>│ 判断!=null       │
  │ StoreLoad屏障     │          │ 使用已初始化对象  │ ← 安全！
  └──────────────────┘          └──────────────────┘
```

## 关联知识点