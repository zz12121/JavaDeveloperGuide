---
title: Buffer 缓冲区
tags:
  - Java/IO
  - 原理型
  - 问答
module: 06_IO与NIO
created: 2026-04-18
---

# Buffer 缓冲区

## Q1：Buffer 的三个核心属性？

**A：**
- **capacity**：容量，分配后固定不变
- **position**：当前位置，下一个读/写的索引
- **limit**：限制，读/写不能超过的索引

关系：`0 ≤ position ≤ limit ≤ capacity`

---

## Q2：flip()、clear()、rewind() 的区别？

**A：**

| 方法 | position | limit | 用途 |
|------|----------|-------|------|
| `flip()` | → 0 | → 原 position | 写完切读 |
| `clear()` | → 0 | → capacity | 清空重写 |
| `rewind()` | → 0 | 不变 | 重新读 |

---

## Q3：ByteBuffer 的使用步骤？

**A：**
1. `ByteBuffer.allocate(1024)` 分配
2. `buffer.put(data)` 写入数据
3. `buffer.flip()` 切换为读模式
4. `buffer.get()` 读取数据
5. `buffer.clear()` 清空，准备再次写入
