---
title: Java对象内存布局
tags:
  - Java/并发
  - 原理型
module: 04_synchronized
created: 2026-04-18
---

# Java对象内存布局（对象头/实例数据/对齐填充）

## 先说结论

Java 对象在 JVM 堆内存中由三部分组成：**对象头（Header）**、**实例数据（Instance Data）**、**对齐填充（Padding）**；其中对象头的 **Mark Word** 是 synchronized 锁机制和 GC 的核心数据结构，其内容会随锁状态动态变化。

## 深度解析

### 1. 对象头（Object Header）

对象头是 JVM 对象最关键的部分，占 **12 字节（64位 JVM，未压缩）** 或 **12 字节（开启指针压缩，Mark Word 8B + Klass Pointer 4B）**，包含以下内容：

#### （1）Mark Word（8 字节）

Mark Word 是对象头中最重要的部分，用于存储对象运行时数据，包括：

| 存储内容 | 说明 |
|---------|------|
| HashCode | 对象的哈希码（调用 `hashCode()` 后延迟写入） |
| GC 年龄 | 对象在 Survivor 区经历过的 GC 次数（4 bit，最大 15） |
| 偏向锁线程 ID | 持有偏向锁的线程 ID |
| 偏向锁时间戳 | 对象首次被偏向的时间 |
| 锁标志位 | 标识当前对象的锁状态（2 bit） |
| 分代年龄标志 | 是否进入老年代的标记 |

> **关键设计思想**：Mark Word 采用了**复用设计**——同一块内存区域在不同锁状态下存储不同的信息，通过锁标志位（最后 2 bit）来区分。这是 JVM 对内存效率的极致优化。

#### （2）Klass Pointer（类型指针，4~8 字节）

指向对象所属类的元数据（`Class` 对象），JVM 通过它确定对象是哪个类的实例。

- **64 位 JVM 未压缩**：8 字节
- **开启指针压缩**（`-XX:+UseCompressedOops`，JDK 6+ 默认开启）：4 字节

#### （3）数组长度（4 字节，仅数组对象）

只有数组类型的对象才包含这一部分，记录数组长度。

### 2. 实例数据（Instance Data）

对象真正存储的有效信息，包括：

- **父类继承的字段**
- **子类自身定义的字段**
- **基本类型**（int 4B、long 8B、boolean 1B 等）
- **引用类型**（未压缩 8B，压缩后 4B）

> **字段排列规则**：JVM 默认按字段宽度递减排列（long/double → int/float → short/char → byte/boolean → 引用），称为 **Monotonic Allocation**，目的是减少内存碎片、提高缓存命中率。可通过 `-XX:FieldsAllocationStyle` 调整。

### 3. 对齐填充（Padding）

- **对齐单位**：8 字节
- **规则**：对象大小必须是 8 字节的整数倍
- **作用**：不是必须的，仅当对象大小不足 8 字节倍数时填充 0 到 7 个字节
- **原因**：便于 CPU 高效读取（内存对齐优化 CPU Cache Line 命中率）

### 64位JVM对象布局图

```
┌──────────────────────────────────────────────────────────────┐
│                      Java Object 内存布局                      │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────────────────────────────┐                │
│  │            Mark Word (8 字节)            │  ← 对象头       │
│  │  ┌───────────────────────────────────┐  │                │
│  │  │ hashcode / gc_age / lock_info     │  │                │
│  │  │ (内容随锁状态动态变化)              │  │                │
│  │  └───────────────────────────────────┘  │                │
│  ├─────────────────────────────────────────┤                │
│  │       Klass Pointer (4/8 字节)          │  ← 对象头       │
│  │       (指向类元数据，开启压缩为4字节)     │                │
│  ├─────────────────────────────────────────┤                │
│  │    Array Length (4字节，仅数组对象)       │  ← 对象头(可选) │
│  ├─────────────────────────────────────────┤                │
│  │                                       │                │
│  │           实例数据 (Instance Data)       │  ← 字段内容     │
│  │     ┌───────┬───────┬───────┬───────┐  │                │
│  │     │ long  │ int   │ short │ byte  │  │                │
│  │     │ 8B    │ 4B    │ 2B    │ 1B    │  │                │
│  │     └───────┴───────┴───────┴───────┘  │                │
│  │                                       │                │
│  ├─────────────────────────────────────────┤                │
│  │           对齐填充 (Padding)             │  ← 凑齐8字节倍数│
│  │         (0 ~ 7 字节的 0x00)            │                │
│  └─────────────────────────────────────────┘                │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Mark Word 结构（64位 JVM）

Mark Word 的 64 bit 在不同锁状态下存储内容不同：

```
                     64 bits
┌─────────────────────────────────────────────────────────────┐
│ unused:25 │ identity_hashcode:31 │ unused:1 │ age:4 │ biased:1 │ lock:2 │  无锁状态
├─────────────────────────────────────────────────────────────┤
│ thread:54 │       epoch:2       │ unused:1 │ age:4 │ biased:1 │ lock:2 │  偏向锁
├─────────────────────────────────────────────────────────────┤
│                  ptr_to_lock_record:62                               │ lock:2 │  轻量级锁
├─────────────────────────────────────────────────────────────┤
│                  ptr_to_heavyweight_monitor:62                         │ lock:2 │  重量级锁
├─────────────────────────────────────────────────────────────┤
│                                                             │ lock:2 │  GC 标记
└─────────────────────────────────────────────────────────────┘
```

**详细状态对照表：**

| 锁状态 | lock 标志位 | biased 位 | 存储内容 | 总 bit |
|-------|-----------|----------|---------|--------|
| 无锁 | 01 | 0 | unused(25) + hashcode(31) + unused(1) + age(4) | 62 |
| 偏向锁 | 01 | 1 | thread ID(54) + epoch(2) + unused(1) + age(4) | 62 |
| 轻量级锁 | 00 | — | 指向锁记录（Lock Record）的指针 | 62 |
| 重量级锁 | 10 | — | 指向 Monitor 对象的指针 | 62 |
| GC 标记 | 11 | — | 空（标记阶段使用） | 62 |

> **面试关键点**：当线程第一次获取偏向锁时，Mark Word 中的 hashcode 会被**覆盖清除**。所以一旦对象计算过 hashcode，就**无法再进入偏向锁状态**，会直接跳到轻量级锁。

## 易错点/踩坑

1. **hashCode 与偏向锁的互斥**：调用 `hashCode()` 后，对象的偏向锁标记被清除，之后只能使用轻量级锁或重量级锁。同理，`System.identityHashCode()` 也会使偏向锁失效。

2. **指针压缩的影响**：开启 `-XX:+UseCompressedOops`（默认开启）后，对象头从 16 字节压缩为 12 字节（Mark Word 8B + Klass Pointer 4B），Klass Pointer 缩减为 4 字节。这在计算对象大小时经常被忽略。

3. **对齐填充不一定存在**：如果实例数据 + 对象头恰好是 8 的倍数，就不需要填充。例如一个只有 `int` 字段的普通对象，大小 = 12（头）+ 4（int）= 16 字节，无需填充。

4. **静态字段不属于对象布局**：`static` 字段存储在类的元数据区（方法区/元空间），不在对象实例中。

5. **Shallow Size vs Retained Size**：JOL 工具打印的是 **Shallow Size**（浅层大小，不含引用对象），对象实际占用内存还需加上引用指向的对象。

## 代码示例

### 使用 JOL 工具查看对象内存布局

**Maven 依赖：**
```xml
<dependency>
    <groupId>org.openjdk.jol</groupId>
    <artifactId>jol-core</artifactId>
    <version>0.17</version>
</dependency>
```

**示例代码：**
```java
import org.openjdk.jol.info.ClassLayout;

public class ObjectLayoutDemo {
    
    // 空对象
    static class EmptyObject {}
    
    // 含基本类型字段
    static class UserObject {
        int id;            // 4 字节
        boolean active;    // 1 字节
        String name;       // 4 字节（压缩指针）
    }
    
    // 数组对象
    static class ArrayObject {
        int[] numbers = new int[10]; // 4(长度) + 40(数据) + 0(已对齐)
    }
    
    public static void main(String[] args) {
        // 打印空对象布局
        System.out.println("=== EmptyObject ===");
        System.out.println(ClassLayout.parseInstance(new EmptyObject()).toPrintable());
        
        // 打印含字段对象布局
        System.out.println("=== UserObject ===");
        System.out.println(ClassLayout.parseInstance(new UserObject()).toPrintable());
        
        // 打印数组布局
        System.out.println("=== int[10] ===");
        System.out.println(ClassLayout.parseInstance(new int[10]).toPrintable());
        
        // 对比不同锁状态下的 Mark Word
        Object lock = new Object();
        System.out.println("=== 无锁状态 ===");
        System.out.println(ClassLayout.parseInstance(lock).toPrintable());
        
        synchronized (lock) {
            System.out.println("=== 轻量级锁状态 ===");
            System.out.println(ClassLayout.parseInstance(lock).toPrintable());
        }
    }
}
```

**预期输出（简化）：**
```
=== EmptyObject ===
com.example.EmptyObject object internals:
 OFFSET  SIZE   TYPE DESCRIPTION               VALUE
      0    12        (object header)           N/A
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

## 图解/流程

### 对象创建到内存分配的全过程

```
          new Object()
              │
              ▼
    ┌─────────────────────┐
    │  类加载检查          │  是否已加载？→ 触发类加载
    │  (已加载则跳过)      │
    └────────┬────────────┘
             │
             ▼
    ┌─────────────────────┐
    │  分配堆内存          │  指针碰撞(规整) / 空闲列表(不规整)
    │  (TLAB 优先)        │  Thread Local Allocation Buffer
    └────────┬────────────┘
             │
             ▼
    ┌─────────────────────┐
    │  初始化零值          │  字段赋默认值 (int=0, boolean=false)
    └────────┬────────────┘
             │
             ▼
    ┌─────────────────────┐
    │  设置对象头          │  Mark Word (hashcode 延迟)
    │                     │  Klass Pointer
    │                     │  Array Length (数组)
    └────────┬────────────┘
             │
             ▼
    ┌─────────────────────┐
    │  执行 <init>        │  执行构造方法，赋实际值
    └─────────────────────┘
```

### 锁升级过程中的 Mark Word 变化

```
   创建对象               首次获取锁              竞争出现               激烈竞争
      │                     │                     │                     │
      ▼                     ▼                     ▼                     ▼
 ┌──────────┐         ┌──────────┐         ┌──────────┐         ┌──────────┐
 │  无锁状态 │ ──偏向──▶│ 偏向锁    │ ──撤销──▶│ 轻量级锁  │ ──膨胀──▶│ 重量级锁  │
 │ lock: 01 │         │ lock: 01 │         │ lock: 00 │         │ lock: 10 │
 │biased: 0 │         │biased: 1 │         │biased: - │         │biased: - │
 │ hashcode │         │ threadID │         │ LockRecord│         │ Monitor  │
 └──────────┘         └──────────┘         └──────────┘         └──────────┘
                         │
                    调用hashCode()
                    或有竞争时
                         │
                         ▼
                    偏向锁撤销，降级为无锁
```

## 关联知识点
