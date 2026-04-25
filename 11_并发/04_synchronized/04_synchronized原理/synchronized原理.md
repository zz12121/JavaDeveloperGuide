---
title: synchronized原理
tags:
  - Java/并发
  - 源码型
module: 04_synchronized
created: 2026-04-18
---

# synchronized原理（Mark Word，monitorenter/monitorexit，ObjectMonitor）

## 先说结论

synchronized 的底层实现依赖 **JVM 对象头的 Mark Word**、**monitorenter/monitorexit 字节码指令**和 **ObjectMonitor 对象**。同步方法通过 ACC_SYNCHRONIZED 标志隐式实现，同步代码块通过 monitorenter/monitorexit 显式实现，最终都指向 ObjectMonitor 的 enter/exit 操作。

## 深度解析

### 对象头 Mark Word

HotSpot JVM 的对象头中 Mark Word 存储了锁相关信息（以 64 位 JVM 为例）：

| 锁状态   | 25bit        | 31bit | 1bit( biased) | 4bit | 1bit( locked) | 2bit |
| -------- | ------------ | ----- | ------------- | ---- | ------------- | ---- |
| 无锁     | unused       | hashCode | 0           | age | 0             | 01   |
| 偏向锁   | thread ID(54)| epoch | 1             | age | 0             | 01   |
| 轻量级锁 | 指向Lock Record栈中指针(62bit) | | | | 00 |
| 重量级锁 | 指向ObjectMonitor指针(62bit) | | | | 10   |
| GC 标记  | 空           | | | | 11            |

### monitorenter / monitorexit

同步代码块编译后：

```text
monitorenter     // 获取锁，尝试将 Mark Word 置为指向 Lock Record
// 临界区代码
monitorexit      // 正常退出，释放锁
// 异常处理
monitorexit      // 异常退出，确保锁释放
```

monitorexit 出现两次：正常退出和异常退出，保证异常时锁也能被释放。

### ObjectMonitor

当锁膨胀为重量级锁时，依赖 ObjectMonitor（C++ 实现）：

```c++
ObjectMonitor {
    _owner        // 持有锁的线程
    _EntryList    // 竞争锁的线程队列（BLOCKED）
    _WaitSet      // 调用 wait() 的线程队列（WAITING）
    _recursions   // 重入次数
    _count        // 等待线程数
}
```

## 易错点/踩坑

- ❌ 以为 synchronized 原理只有 monitorenter/monitorexit
- ✅ 同步方法用 ACC_SYNCHRONIZED 标志，不需要字节码指令
- ❌ 以为 ObjectMonitor 在加锁时就创建
- ✅ ObjectMonitor 在锁膨胀为重量级锁时才创建
- ❌ 以为 Mark Word 只有锁信息
- ✅ Mark Word 还存储 hashCode、GC 年龄等

## 代码示例

```java
// 查看字节码：javap -v SyncDemo.class
public class SyncDemo {
    public void method() {
        synchronized (this) {
            System.out.println("hello");
        }
    }
}

// 字节码：
// monitorenter
// getstatic java/lang/System.out
// ldc "hello"
// invokevirtual println
// monitorexit
// goto ...
// astore_1
// monitorexit  // 异常路径也释放锁
// aload_1
// athrow
```

## 图解/流程

```
synchronized 执行流程：
  线程进入同步块
       │
       ▼
  monitorenter
       │
       ▼
  读取对象 Mark Word
       │
       ├── 锁标志=01（无锁/偏向锁）──▶ 偏向锁/轻量级锁路径
       │
       ├── 锁标志=00（轻量级锁）──▶ CAS 竞争 Lock Record
       │
       └── 锁标志=10（重量级锁）──▶ 进入 ObjectMonitor
                │
                ├── _owner == null ──▶ 获取锁，设 _owner = 当前线程
                │
                └── _owner == 其他线程 ──▶ 加入 _EntryList，线程 BLOCKED
                                                        │
                                                  锁释放后被唤醒
                                                        │
                                                  重新竞争锁
```

## 关联知识点
