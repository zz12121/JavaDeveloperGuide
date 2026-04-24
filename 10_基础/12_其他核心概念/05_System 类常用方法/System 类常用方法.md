---
title: System类常用方法
tags:
  - Java/其他核心概念
  - 原理型
module: 12_其他核心概念
created: 2026-04-18
---

## 核心结论

System 是一个 final 类，不能被继承和实例化，提供系统级功能，常用方法包括数组拷贝、获取时间、垃圾回收建议、获取/设置系统属性等。

---

## 深度解析

### 1. 常用方法

#### arraycopy — 数组拷贝
```java
System.arraycopy(src, srcPos, dest, destPos, length);
// src: 源数组
// srcPos: 源数组起始位置
// dest: 目标数组
// destPos: 目标数组起始位置
// length: 拷贝长度

int[] src = {1, 2, 3, 4, 5};
int[] dest = new int[5];
System.arraycopy(src, 0, dest, 0, 5); // dest = [1, 2, 3, 4, 5]
```

#### 时间相关
```java
System.currentTimeMillis();  // 当前时间戳（毫秒）
System.nanoTime();           // 高精度时间（纳秒），用于计时
System.exit(0);              // 退出 JVM（0 正常，非0 异常）
```

#### 垃圾回收
```java
System.gc();         // 建议 JVM 进行垃圾回收（不保证立即执行）
System.runFinalization(); // 建议执行待终结对象的 finalize 方法（已废弃）
```

#### 系统属性
```java
System.getProperty("java.version");   // Java 版本
System.getProperty("user.home");      // 用户目录
System.getProperty("file.separator"); // 文件分隔符
System.setProperty("key", "value");   // 设置系统属性
```

#### 环境变量
```java
System.getenv();             // 所有环境变量
System.getenv("JAVA_HOME");  // 获取指定环境变量
```

### 2. System.arraycopy 性能

| 方法 | 性能 | 说明 |
|------|------|------|
| `System.arraycopy` | ⭐⭐⭐⭐⭐ | native 实现，最快 |
| `Arrays.copyOf` | ⭐⭐⭐⭐ | 底层调用 arraycopy |
| `clone()` | ⭐⭐⭐ | 创建新数组再 arraycopy |
| for 循环 | ⭐⭐ | Java 层面循环赋值 |

### 3. 标准流

```java
System.out  // 标准输出（PrintStream）
System.err  // 标准错误（PrintStream）
System.in   // 标准输入（InputStream）
```

---

## 关联知识点
