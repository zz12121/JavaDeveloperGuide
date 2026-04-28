
# CompactStrings

## Q1：什么是 CompactStrings？

**A**：CompactStrings 是 JDK9 引入的字符串优化：
- **纯 ASCII/LATIN1 字符**：使用 1 字节编码（LATIN1）
- **包含非 ASCII 字符**：使用 2 字节编码（UTF-16）
- 减少字符串内存占用约 30-50%

---

## Q2：JDK8 和 JDK9+ 字符串内部有什么区别？

**A**：

| JDK 版本 | 内部存储 | 编码 | 字符占用 |
|----------|----------|------|----------|
| JDK8 | `char[] value` | UTF-16 | 2 bytes/char |
| JDK9+ | `byte[] value` | LATIN1/UTF16 | 1 或 2 bytes/char |

---

## Q3：哪些字符使用 LATIN1 编码？

**A**：LATIN1 编码范围：
- **ASCII 字符**：0-127
- **西欧字符**：128-255（法语、德语带变音符号等）

不包括：中文、日文、韩文等非拉丁字符（需要 UTF-16）

---

## Q4：CompactStrings 如何节省内存？

**A**：以 "hello world"（11个字符）为例：
| JDK 版本 | 内存计算 | 总大小 |
|----------|----------|--------|
| JDK8 | 11 × 2 + 16(header) + 数组开销 | 50 bytes |
| JDK9+ | 11 × 1 + 16(header) + 编码开销 | 32 bytes |

**节省约 36%**

---

## Q5：如何关闭 CompactStrings？

**A**：JVM 参数：
```bash
-XX:-CompactStrings
```
不推荐关闭，因为：
- LATIN1 字符串仍然是主流
- 混合编码对性能影响很小

---

## Q6：混合编码会影响字符串操作性能吗？

**A**：影响很小：
- String 类内部已经优化了编码选择
- 大部分字符串操作（Java 代码）仍然按字符处理
- JIT 编译器会进一步优化

---

## Q7：中文字符串会使用哪种编码？

**A**：中文字符**不使用 LATIN1**，因为：
- 中文字符 Unicode 码点 > 255
- 会触发 UTF-16 编码
- UTF-16 下，中文和英文占用相同空间
