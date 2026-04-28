
# StringTable

## 先说结论

**StringTable（字符串常量池）** 是一个 HashTable，存储字符串字面量和 `intern()` 的字符串引用。JDK8 中位于堆中，大小可通过 `-XX:StringTableSize` 参数调整。

## 深度解析

### StringTable 位置与结构

```
┌──────────────────────────────────────────────────────────────────┐
│                    StringTable（字符串常量池）                     │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  JDK8 位置：堆中                                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                         堆（Heap）                          │ │
│  │  ┌──────────────────────────────────────────────────────┐  │ │
│  │  │               StringTable (HashTable)                 │  │ │
│  │  │  ┌─────────┬─────────┬─────────┬─────────┬────────┐  │  │ │
│  │  │  │ "hello" │ "world" │ "java"  │ intern()│  ...   │  │  │ │
│  │  │  │ (ref)   │ (ref)   │ (ref)   │ (ref)   │        │  │  │ │
│  │  │  └─────────┴─────────┴─────────┴─────────┴────────┘  │  │  │
│  │  └──────────────────────────────────────────────────────┘  │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### StringTable vs 运行时常量池

| 特性 | StringTable | 运行时常量池 |
|------|-------------|-------------|
| **存储内容** | 字符串 | 各种常量（int、long、Method等） |
| **位置** | 堆中（JDK8+） | 方法区/元空间 |
| **唯一性** | 字符串唯一 | 常量唯一 |
| **大小调整** | `-XX:StringTableSize` | 固定 |

### StringTable 原理

```java
// intern() 原理
public String intern() {
    // 1. 检查 StringTable 是否有当前字符串
    // 2. 如果有，返回池中的引用
    // 3. 如果没有，加入 StringTable，返回引用
}
```

### 配置参数

```bash
# StringTable 大小（JDK8+）
# 默认：60013（JDK8u102+）
# 建议：设置为一个素数，减少哈希冲突
-XX:StringTableSize=200000

# 查看 StringTable 统计
jcmd <pid> VM.stringtable
```

## 易错点/踩坑

- ❌ StringTable 大小影响 intern() 性能
- ❌ StringTable 是共享的，存在并发竞争
- ❌ intern() 调用不当可能导致性能问题
- ❌ StringTable 和 运行时常量池 是不同的结构

## 代码示例

```java
public class StringTableDemo {
    public static void main(String[] args) {
        // 字面量自动入池
        String s1 = "hello";
        String s2 = "hello";
        System.out.println(s1 == s2);  // true
        
        // intern() 手动入池
        String s3 = new String("world");
        String s4 = s3.intern();
        System.out.println(s3 == s4);  // false
        System.out.println(s4 == "world");  // true
    }
}
```

```bash
# 查看 StringTable 统计
$ jcmd 12345 VM.stringtable
# 输出
StringTable statistics:
Number of buckets: 60013
Number of entries: 1523
Longest chain: 7

# -XX:StringTableSize=200000 设置更大的表
java -XX:StringTableSize=200000 YourApp
```

## 图解/流程

```
┌──────────────────────────────────────────────────────────────────┐
│                    StringTable 工作流程                           │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  String s = "hello"                                               │
│         ↓                                                         │
│  计算 hash("hello") → hashcode                                    │
│         ↓                                                         │
│  定位 hash bucket                                                 │
│         ↓                                                         │
│   ┌─────────┴─────────┐                                           │
│   ↓                   ↓                                           │
│  bucket有           bucket空                                       │
│   ↓                   ↓                                           │
│  遍历链表查找        直接插入                                       │
│   ↓                   ↓                                           │
│  ┌─────────────────┐  ┌─────────────────┐                        │
│  │ 已存在？返回ref  │  │ 不存在？新建并插入│                        │
│  └─────────────────┘  └─────────────────┘                        │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

## 关联知识点
