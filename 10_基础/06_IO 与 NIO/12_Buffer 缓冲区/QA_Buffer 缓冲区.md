---
title: Buffer 缓冲区面试题
tags:
  - Java/IO
  - 原理型
  - 问答
module: 06_IO与NIO
created: 2026-04-18
updated: 2026-04-25
---

# Buffer 缓冲区

## Q1：Buffer 的三个核心属性是什么？它们的关系？

**A：**

| 属性 | 含义 |
|------|------|
| **capacity（容量）** | 缓冲区大小，分配后固定不变 |
| **position（位置）** | 下一个读/写操作的索引 |
| **limit（限制）** | 读/写操作不能超过的索引 |

三者关系：**`0 ≤ position ≤ limit ≤ capacity`**

```
初始状态 [allocate(10)]：
┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
│   │   │   │   │   │   │   │   │   │   │
└───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
0                                              10
p=0              l=10            c=10
```

---

## Q2：flip()、clear()、rewind() 的区别？

**A：**

| 方法 | position | limit | mark | 用途 |
|------|:---:|:---:|:---:|------|
| `flip()` | → 0 | → 原 position | -1 | 写完切换为读模式 |
| `clear()` | → 0 | → capacity | -1 | 清空，准备重新写入 |
| `rewind()` | → 0 | 不变 | -1 | 重新读一遍（不改变 limit） |

```
写入 "abc" 后执行 flip()：
┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
│ a │ b │ c │   │   │   │   │   │   │   │
└───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
0       3       10
p=0     l=3     c=10   ← limit 限制了只能读到索引 3
```

**常见错误**：读完后直接写，要先 `clear()` 而不是直接 `put()`。

---

## Q3：compact() 方法的作用？

**A：**

`compact()` 是**保留未读数据 + 切换写模式**的组合操作：

```java
public Buffer compact() {
    System.arraycopy(buffer, position, buffer, 0, remaining());
    position = remaining();
    limit = capacity;
    mark = -1;
    return this;
}
```

场景：读了一部分数据，但还有一部分没读完，此时想继续写入新数据。

```
compact() 前（已读 3 字节，共写了 6 字节）：
┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
│ d │ e │ f │ c │ d │ e │   │   │   │   │
└───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
0       3       6                      10
        p=3     l=6            c=10

compact() 后（未读的 cde 移到开头，继续写入）：
┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
│ c │ d │ e │   │   │   │   │   │   │   │
└───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
0       3           10
p=3             l=10 c=10
```

**与 clear() 的区别**：`clear()` 丢弃所有数据，`compact()` 保留未读数据。

---

## Q4：mark() 和 reset() 的作用？

**A：**

```java
buffer.mark();   // 记录当前 position（保存书签）
buffer.get();    // 读取一些数据，position 变化
buffer.get();
buffer.reset();  // 将 position 恢复到 mark 位置（重新读）
```

注意：`flip()` 和 `clear()` 会自动将 mark 置为 -1（取消书签）。

---

## Q5：remaining() 和 hasRemaining() 的区别？

**A：**

```java
int remaining();      // 返回 limit - position（剩余可读/可写字节数）
boolean hasRemaining(); // 返回 position < limit（是否还有剩余）
```

使用场景：
```java
while (buffer.hasRemaining()) {
    channel.read(buffer);  // 或 channel.write(buffer)
}
```

---

## Q6：ByteBuffer 的使用完整步骤？

**A：**

```java
// 1. 分配
ByteBuffer buffer = ByteBuffer.allocate(1024);
// ByteBuffer buffer = ByteBuffer.allocateDirect(1024);

// 2. 写入数据
buffer.put("Hello".getBytes());

// 3. 切换为读模式
buffer.flip();

// 4. 读取数据
byte[] data = new byte[buffer.remaining()];
buffer.get(data);

// 5. 清空，准备再次写入
buffer.clear();  // 注意：clear() 后数据未真正删除，只是 position=0

// 如果读了一部分还想继续写：用 compact() 代替 clear()
```

---

## Q7：HeapByteBuffer 和 DirectByteBuffer 在 Buffer 操作上有何不同？

**A：**

| 操作 | HeapByteBuffer | DirectByteBuffer |
|------|:---:|:---:|
| 底层数组 | `byte[] hb`（JVM 堆内） | `long address`（堆外内存地址） |
| 分配 | `allocate()` | `allocateDirect()` |
| 读写效率 | 较快（堆内访问） | 更快（避免堆外拷贝） |
| I/O 时 | 需要拷贝到直接内存 | 直接用于系统调用 |

底层实现差异：

```java
// HeapByteBuffer — put 操作直接写入堆数组
public ByteBuffer put(byte x) {
    hb[index++] = x;  // 直接写入堆内存数组
    return this;
}

// DirectByteBuffer — put 操作写入堆外内存地址
private native void put(long addr, byte b);  // JNI 调用操作堆外内存
```

两者的 `flip()`、`clear()`、`rewind()` 核心逻辑是相同的，只是底层存储位置不同。
