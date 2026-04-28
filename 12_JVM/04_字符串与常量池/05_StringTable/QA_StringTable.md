
# StringTable

## Q1：StringTable 是什么？

**A**：StringTable（字符串常量池）是 JVM 管理的 HashTable，用于存储字符串字面量和 `intern()` 的字符串引用。JDK8 中位于堆中。

---

## Q2：StringTable 和 String Pool 是同一个吗？

**A**：是的，StringTable 就是 String Pool（字符串常量池）。在不同 JDK 版本中：
- JDK6：位于永久代
- JDK7+：位于堆中

---

## Q3：StringTable 如何存储字符串？

**A**：StringTable 是一个 HashTable：
- **bucket**：计算字符串 hash 定位 bucket
- **链表**：解决哈希冲突（链地址法）
- **去重**：相同字符串只存一份

---

## Q4：String s = "hello" 和 new String("hello") 的区别？

**A**：

| 方式 | 是否入池 | 创建对象 |
|------|----------|----------|
| `"hello"` | 是 | 可能不创建（已存在则直接返回引用） |
| `new String("hello")` | 否（堆对象） | 1-2 个对象 |

---

## Q5：StringTable 大小如何调整？

**A**：`StringTableSize` 参数：
```bash
# 默认 60013（JDK8u102+）
-XX:StringTableSize=200000

# 建议设置为素数，减少哈希冲突
```

---

## Q6：intern() 会查找 StringTable 吗？

**A**：会。`intern()` 流程：
1. 检查 StringTable 是否有该字符串
2. **有**：返回 StringTable 中的引用
3. **没有**：将字符串加入 StringTable，返回引用

---

## Q7：StringTable 和运行时常量池的关系？

**A**：
- **StringTable**：只存储字符串
- **运行时常量池**：存储各种类型的常量（int、long、Method等）
- **关系**：StringTable 可以看作是运行时常量池中字符串相关内容的具体实现

---

## Q8：StringTable 会 OOM 吗？

**A**：会。JDK8 中：
- StringTable 位于堆中
- 如果大量 intern() 但不回收，可能 OOM
- 但通常 GC 会回收不再使用的字符串

