
# JDK9 CompactStrings

## 先说结论

JDK9 引入 **CompactStrings**，对纯 ASCII 字符的字符串使用 **Latin-1 编码**（1 byte/char），对包含非 ASCII 字符的字符串使用 **UTF-16 编码**（2 bytes/char）。有效减少字符串内存占用。

## 深度解析

### 编码方案对比

| JDK 版本 | 编码方式 | 字符占用 | 说明 |
|----------|----------|----------|------|
| JDK8 | UTF-16 | 2 bytes/char | 固定 |
| JDK9+ | Latin-1 + UTF-16 | 1 或 2 bytes/char | 动态选择 |

### CompactStrings 原理

```
┌──────────────────────────────────────────────────────────────────┐
│                    JDK9+ 字符串编码                                │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  String 内部结构                                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  char[] value;  →  byte[] value; (JDK9)                   │ │
│  │                      ↓                                     │ │
│  │              coder 字段标识编码方式                          │ │
│  │              - LATIN1 (0)                                  │ │
│  │              - UTF16  (1)                                  │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                   │
│  字符串 "hello" (纯 ASCII)                                         │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  coder = LATIN1                                            │ │
│  │  value = [h][e][l][l][o]  // 每个1字节                      │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                   │
│  字符串 "你好" (非ASCII)                                           │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  coder = UTF16                                             │ │
│  │  value = [你][好]  // 每个2字节                            │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### 内存节省

| 字符串内容 | JDK8 内存 | JDK9+ 内存 | 节省 |
|-----------|-----------|-----------|------|
| "hello" | 50 bytes | 32 bytes | 36% |
| "你好" | 38 bytes | 38 bytes | 0% |
| "a"×100 | 208 bytes | 112 bytes | 46% |

### 编码选择规则

```java
// String 类内部
private final byte[] value;
private final byte coder; // 0 = LATIN1, 1 = UTF16

// 选择编码逻辑
if (全部是 LATIN1 可表示字符) {
    使用 LATIN1 编码;
} else {
    使用 UTF16 编码;
}
```

## 易错点/踩坑

- ❌ CompactStrings 不是替换编码，而是**混合编码**
- ❌ 非 ASCII 字符（如中文）仍使用 UTF-16
- ❌ StringBuilder/Buffer 内部也使用同样优化
- ❌ 关闭 CompactStrings：`-XX:-CompactStrings`

## 代码示例

```java
public class CompactStringDemo {
    public static void main(String[] args) {
        // 纯 ASCII：使用 LATIN1
        String ascii = "hello world";
        
        // 包含非 ASCII：使用 UTF16
        String chinese = "你好";
        
        // 混合：使用 UTF16
        String mixed = "hello你好";
        
        // 查看编码方式（需要反射）
        // getDeclaredField("value") 和 ("coder")
    }
}
```

```bash
# JVM 参数
-XX:-CompactStrings  # 关闭 CompactStrings

# 查看字符串对象大小
java -cp jol-core.jar \
    org.openjdk.jol.info.ClassLayout -XX:-CompactStrings .

# 对比开关效果
java -cp jol-core.jar \
    org.openjdk.jol.info.ObjectHistogram .
```

## 图解/流程

```
┌──────────────────────────────────────────────────────────────────┐
│                    CompactStrings 选择流程                        │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  创建 String                                                       │
│       ↓                                                            │
│  ┌─────────────────────────────────────────┐                     │
│  │  检查字符串是否只包含 LATIN1 字符？        │                     │
│  │  (LATIN1: 0-255, 即 ASCII + 西欧字符)     │                     │
│  └─────────────────┬───────────────────────┘                     │
│          是        ↓              否                               │
│          ↓    LATIN1         ↓  UTF16                             │
│    ┌─────────────┐    ┌─────────────┐                            │
│    │ byte[1/len] │    │ byte[2/len] │                            │
│    │ coder = 0   │    │ coder = 1   │                            │
│    └─────────────┘    └─────────────┘                            │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

## 关联知识点
