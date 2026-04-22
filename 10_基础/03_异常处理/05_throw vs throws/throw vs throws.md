---
title: throw vs throws
tags:
  - Java/异常处理
  - 对比型
module: 03_异常处理
created: 2026-04-18
---

# throw vs throws

## 核心对比

| 对比项 | `throw` | `throws` |
|--------|---------|----------|
| 位置 | 方法体内 | 方法签名上 |
| 作用 | **主动抛出**一个异常实例 | **声明**方法可能抛出的异常类型 |
| 后面跟 | 异常对象（实例） | 异常类（可多个，逗号分隔） |
| 数量 | 一次只能抛一个 | 可声明多个 |
| 执行时机 | 运行时执行到该语句 | 编译期检查 |

## 代码示例

```java
// throws：声明可能抛出的异常（方法签名）
public void readFile(String path) throws IOException, IllegalArgumentException {
    if (path == null) {
        throw new IllegalArgumentException("path 不能为 null");  // throw：实际抛出
    }
    FileInputStream fis = new FileInputStream(path);  // 可能抛 IOException
}
```

```java
// throw 主动抛出
public void checkAge(int age) {
    if (age < 0) {
        throw new IllegalArgumentException("年龄不能为负数：" + age);
    }
    if (age > 150) {
        throw new IllegalArgumentException("年龄超过合理范围：" + age);
    }
}
```

## 规则说明

### throw 规则
1. throw 后面必须跟 `Throwable` 的实例（或子类）
2. throw 之后不能有其他语句（编译器认为后面代码不可达）
3. 抛出 Checked Exception → 方法必须声明 throws 或在 try-catch 中处理

### throws 规则
1. 声明 Checked Exception → 调用方必须处理（try-catch 或继续 throws）
2. 声明 Unchecked Exception → 可以不处理（仅起文档说明作用）
3. 子类重写方法时，throws 声明的异常范围不能比父类更宽

## 子类重写的 throws 规范

```java
class Parent {
    public void method() throws IOException {}
}

class Child extends Parent {
    // ✅ 合法：抛出更具体的子类异常
    public void method() throws FileNotFoundException {}

    // ✅ 合法：不抛出异常
    // public void method() {}

    // ❌ 非法：抛出更宽泛的异常（编译报错）
    // public void method() throws Exception {}
}
```

---

## 关联知识点

