---
title: transient关键字面试题
tags:
  - Java/IO
  - 原理型
  - 问答
module: 06_IO与NIO
created: 2026-04-25
---

# transient关键字

## Q1：transient 修饰的字段为什么不会被序列化？

**A：**
`transient` 修饰的字段在序列化时被**跳过**（跳过写入）。JVM 序列化机制会忽略 transient 字段：

```java
class User implements Serializable {
    String username;        // 被序列化
    transient String password;  // 跳过，不被序列化
    transient int cache;        // 跳过，不被序列化
}
```

序列化后的 User 对象中，password 和 cache 字段为默认值（null/0），而不是实际值。

---

## Q2：transient 和 static 字段有什么区别？

**A：**

| 字段 | 序列化行为 | 反序列化结果 |
|------|----------|------------|
| 普通字段 | 序列化 | 恢复实际值 |
| `static` 字段 | **不序列化**（类信息，不属于对象） | JVM 默认值 |
| `transient` 字段 | **跳过**（属于对象，但不保存） | JVM 默认值 |

两者都不参与序列化，但原因不同：
- `static`：属于类，不属于对象
- `transient`：属于对象，但被显式排除

---

## Q3：transient 的典型使用场景？

**A：**
1. **敏感数据**：密码、密钥、token 等不应该持久化的字段
2. **计算派生字段**：可以从其他字段推导出来的字段
3. **临时缓存**：不需要持久化的缓存数据
4. **非序列化引用**：引用了不可序列化的对象

```java
class Order implements Serializable {
    String orderId;                    // 序列化
    double price;                     // 序列化
    transient String cachedSignature;  // 跳过（临时计算）
    transient Connection dbConn;        // 跳过（不可序列化）
    transient Map<String, Object> tempCache;  // 跳过（临时缓存）
}
```

---

## Q4：如何让 transient 字段也能被序列化？

**A：**
实现 `writeObject` 和 `readObject` 自定义序列化逻辑：

```java
class User implements Serializable {
    String username;
    transient String password;

    // 自定义序列化
    private void writeObject(ObjectOutputStream out) throws IOException {
        out.defaultWriteObject();       // 先序列化非 transient 字段
        out.writeObject(encrypt(password));  // 手动序列化并加密
    }

    private void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException {
        out.defaultReadObject();        // 先反序列化非 transient 字段
        password = decrypt((String) in.readObject());  // 手动反序列化并解密
    }
}
```

---

## Q5：transient 字段的反序列化默认值是什么？

**A：**
| 字段类型 | 反序列化默认值 |
|---------|------------|
| 基本类型 | 0/false |
| 对象引用 | null |
| 数组 | null |
| Collection | null |
| Map | null |

即使序列化前有值，反序列化后都变成默认值，这是 transient 的主要风险。
