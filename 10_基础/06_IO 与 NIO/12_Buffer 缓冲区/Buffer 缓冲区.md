---
id: card_79
title: Buffer 缓冲区（flip / clear / rewind）
tags:
  - Java/IO
  - 卡片
  - 常考
  - 原理型
module: "06_IO与NIO"
created: 2026-04-18
---

# Buffer 缓冲区（flip / clear / rewind）

## Buffer 核心属性

```java
public abstract class Buffer {
    private int position;  // 当前位置（下一个读/写的索引）
    private int limit;     // 限制（不能超过的索引）
    private int capacity;  // 容量（固定不变）
}
```

```
初始状态（allocate(10)）：
┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
│   │   │   │   │   │   │   │   │   │   │
└───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
0                                              10
p=0              l=10           c=10
```

## 状态转换

### 写入数据后

```
put("abc") 后：
┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
│ a │ b │ c │   │   │   │   │   │   │   │
└───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
0       3                                  10
                p=3           l=10  c=10
```

### flip() —— 写模式 → 读模式

```java
public final Buffer flip() {
    limit = position;  // limit 设为当前 position
    position = 0;      // position 归零
    mark = -1;
    return this;
}
```

```
flip() 后：
┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
│ a │ b │ c │   │   │   │   │   │   │   │
└───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
0       3       10
p=0     l=3     c=10
```

### clear() —— 清空缓冲区

```java
public final Buffer clear() {
    position = 0;
    limit = capacity;
    mark = -1;
    return this;
}
```

```
clear() 后：回到初始状态，准备重新写入
p=0     l=10    c=10
```

### rewind() —— 重置 position（不改变 limit）

```java
public final Buffer rewind() {
    position = 0;
    mark = -1;
    return this;
}
```

用于：已经读过一部分，想**重新读一遍**。

## 三个方法对比

| 方法 | position | limit | 用途 |
|------|----------|-------|------|
| `flip()` | → 0 | → 原 position | 写完后切换为读模式 |
| `clear()` | → 0 | → capacity | 清空，准备重新写 |
| `rewind()` | → 0 | 不变 | 重新读（不改变数据） |

## 常用 Buffer 类型

| 类型 | 说明 |
|------|------|
| `ByteBuffer` | 字节（最常用） |
| `CharBuffer` | 字符 |
| `IntBuffer` | int |
| `LongBuffer` | long |
| `FloatBuffer` | float |
| `DoubleBuffer` | double |
| `ShortBuffer` | short |
| `MappedByteBuffer` | 内存映射文件 |

## 使用模板

```java
// 1. 分配
ByteBuffer buffer = ByteBuffer.allocate(1024);

// 2. 写入
buffer.put("Hello".getBytes());

// 3. 切换为读模式
buffer.flip();

// 4. 读取
byte[] data = new byte[buffer.remaining()];
buffer.get(data);

// 5. 清空（准备再次写入）
buffer.clear();
```

## 关联知识点

