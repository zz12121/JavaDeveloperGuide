
# new String 创建对象数

## 先说结论

`new String("abc")` 会创建 **1 或 2 个对象**：1 个是堆中的 String 对象，1 个是字符串常量池中的字符串（如果常量池没有）。字符串拼接 `new String("a") + new String("b")` 会创建 **多个对象**。

## 深度解析

### new String("abc") 解析

```java
// 字节码分析
NEW java/lang/String
DUP
LDC "abc"              // 从常量池加载 "abc"
INVOKESPECIAL java/lang/String.<init>
```

**创建对象**：
1. **1 个堆对象**：new String()
2. **1 个字符串常量**：如果常量池没有 "abc"，先创建

### 字符串拼接 new String 分析

```java
String s = new String("a") + new String("b");
```

**字节码解析**：
```
1. NEW StringBuilder          // StringBuilder 对象
2. 编译优化后拼接
3. 调用 toString() 
```

**创建对象**：
1. `new String("a")` → 1-2 个对象（看常量池）
2. `new String("b")` → 1-2 个对象（看常量池）
3. StringBuilder → 1 个对象
4. `new String()` → toString() 产生的堆对象

**总计**：至少 **5-6 个对象**

### 对象数量分析表

| 代码 | 创建对象数 | 说明 |
|------|-----------|------|
| `"abc"` | 0 或 1 | 字面量已存在则不创建 |
| `new String("abc")` | 1 或 2 | 1个堆对象 + 可能1个常量池对象 |
| `new String(chars)` | 1 | 直接用字符数组 |
| `"a" + "b"` | 0 | 编译期常量折叠为 "ab" |

### 经典面试题

```java
// 问：创建了几个 String 对象？
String s1 = new String("hello");
String s2 = "hello";

// 答案：1个或2个
// - s1：堆中1个
// - 如果常量池没有 "hello"：再创建1个

// 问：创建了几个 String 对象？
String s = new String("a") + new String("b");

// 答案：至少5个
// 1. new String("a") - 堆对象
// 2. "a" - 常量池对象
// 3. new String("b") - 堆对象  
// 4. "b" - 常量池对象
// 5. StringBuilder - 拼接用
// 6. s.toString() 产生的 "ab" 对象（堆）
```

## 易错点/踩坑

- ❌ `new String("abc")` 不一定创建 2 个对象
- ❌ 字符串字面量在编译期就确定
- ❌ `"a" + "b"` 在编译期优化为 "ab"
- ❌ StringBuilder 的 toString() 创建新对象

## 代码示例

```java
public class StringObjectCount {
    public static void main(String[] args) {
        // 案例1
        String s1 = "hello";           // 字面量，可能不创建对象
        String s2 = new String("hello"); // 1-2个对象
        
        // 案例2：intern 影响对象数量
        String s3 = new String("hello");
        s3.intern();                    // 将 "hello" 加入常量池
        String s4 = "hello";            // 直接从常量池获取
        
        // 案例3：编译期优化
        String s5 = "a" + "b";          // 编译期优化为 "ab"
        String s6 = "ab";
        System.out.println(s5 == s6);  // true
        
        // 案例4：变量拼接（不优化）
        String a = "a";
        String s7 = a + "b";           // 运行时 StringBuilder
        System.out.println(s7 == s6);  // false
    }
}
```

## 图解/流程

```
new String("abc") 执行流程

1. 检查常量池是否有 "abc"
         ↓
   ┌─────────┴─────────┐
   ↓                   ↓
 常量池有            常量池没有
   ↓                   ↓
 不创建常量池对象    先在常量池创建 "abc"
         ↓                   ↓
   创建1个堆对象      创建2个对象
```

## 关联知识点
