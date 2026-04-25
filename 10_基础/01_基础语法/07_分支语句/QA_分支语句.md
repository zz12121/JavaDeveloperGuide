---
title: "分支语句 - 面试题"
tags:
  - Java
  - 基础语法
  - 流程控制
module: 01_基础语法
created: 2026-04-25
---

# Q&A - 分支语句

## Q1: if-else 和 switch 的区别？各自适用场景？

**A：**

### 核心区别

| 方面 | if-else | switch |
|------|---------|--------|
| **条件类型** | 任意 boolean 表达式 | 等值判断（整数、字符、枚举、String） |
| **适用场景** | 范围判断、复合条件 | 固定值匹配 |
| **分支数量** | 任意 | 通常 3-10 个 |
| **执行效率** | 顺序判断 O(n) | 跳转表 O(1) 或 HashMap |
| **可读性** | 嵌套时差 | 分支多时更好 |

### if-else 适用场景

```java
// 1. 范围判断
if (score >= 90) {
    grade = "A";
} else if (score >= 80) {
    grade = "B";
} else if (score >= 70) {
    grade = "C";
} else {
    grade = "D";
}

// 2. 复合条件
if (age >= 18 && hasLicense && !isSuspended) {
    // 复杂布尔表达式
}

// 3. 边界条件检查
if (value == null || value < 0) {
    throw new IllegalArgumentException();
}
```

### switch 适用场景

```java
// 1. 固定值匹配
switch (day) {
    case "MONDAY":    // ...
    case "TUESDAY":   // ...
    case "WEDNESDAY": // ...
}

// 2. 枚举类型
switch (status) {
    case PENDING:  // ...
    case APPROVED: // ...
    case REJECTED: // ...
}

// 3. 状态机
switch (state) {
    case IDLE:    // 空闲状态
    case RUNNING: // 运行状态
    case PAUSED:  // 暂停状态
}
```

### 性能对比

```java
// if-else：顺序判断
if (value == 1) {
    // 第1次判断
} else if (value == 2) {
    // 第2次判断
} else if (value == 3) {
    // 第3次判断
}
// 最坏情况：3次比较

// switch：编译器优化
// JVM 实现：
// 1. 少分支（<=4）：类似 if-else
// 2. 多分支：生成跳转表（tableswitch）或二分查找（lookupswitch）
// 平均：O(1)
```

### 选择建议

```
选择 if-else 的场景：
├─ 需要范围判断（score >= 90）
├─ 条件是布尔表达式（age > 18 && hasLicense）
└─ 分支少于 3 个

选择 switch 的场景：
├─ 条件是离散值匹配（day == "MONDAY"）
├─ 分支数量 3-10 个
└─ 条件可穷举（枚举、有限状态）
```

---

## Q2: switch 支持哪些数据类型？（JDK 5 支持枚举，JDK 7 支持 String，JDK 14+ 支持表达式）

**A：**

### 支持的数据类型

| 类型 | 支持版本 | 说明 |
|------|---------|------|
| `byte` | JDK 1.0+ | 8位整数 |
| `short` | JDK 1.0+ | 16位整数 |
| `int` | JDK 1.0+ | 32位整数 |
| `char` | JDK 1.0+ | 16位Unicode字符 |
| `枚举 (enum)` | JDK 5.0+ | 枚举类型 |
| `String` | JDK 7.0+ | 字符串（JDK 8+ 支持在 lambda） |

### 不支持的数据类型

```java
// ❌ long - 编译错误
long l = 1;
switch (l) {  // Error: incompatible types
    case 1: break;
}

// ❌ float - 编译错误
float f = 1.0f;
switch (f) {  // Error: incompatible types
    case 1.0f: break;
}

// ❌ double - 编译错误
double d = 1.0;
switch (d) {  // Error: incompatible types
    case 1.0: break;
}

// ❌ boolean - 编译错误
boolean b = true;
switch (b) {  // Error: incompatible types
    case true: break;
}
```

### 各类型示例

```java
// byte
byte grade = 2;
switch (grade) {
    case 1: System.out.println("一年级"); break;
    case 2: System.out.println("二年级"); break;
}

// short
short month = 3;
switch (month) {
    case 1: case 2: case 3: System.out.println("第一季度"); break;
    // ...
}

// int
int dayOfWeek = 3;
switch (dayOfWeek) {
    case 1: System.out.println("周一"); break;
    case 2: System.out.println("周二"); break;
    // ...
}

// char
char grade = 'A';
switch (grade) {
    case 'A': System.out.println("优秀"); break;
    case 'B': System.out.println("良好"); break;
}

// enum
enum Status { PENDING, APPROVED, REJECTED }
Status status = Status.APPROVED;
switch (status) {
    case PENDING: System.out.println("待处理"); break;
    case APPROVED: System.out.println("已批准"); break;
    case REJECTED: System.out.println("已拒绝"); break;
}

// String (JDK 7+)
String role = "admin";
switch (role) {
    case "admin": System.out.println("管理员"); break;
    case "user": System.out.println("普通用户"); break;
    case "guest": System.out.println("访客"); break;
}
```

### 为什么 long/float/double/boolean 不支持？

```java
// 1. long：范围太大
// switch 内部使用跳转表，long 无法高效索引
// 替代方案：使用 if-else 或 Long.compare()

// 2. float/double：精度问题
// 浮点数比较需要考虑精度，switch 的精确匹配不适用
// 替代方案：使用 if-else 或 BigDecimal

// 3. boolean：只有两个值
// if-else 更简洁直接
// if (flag) { } else { }
```

---

## Q3: switch 表达式 vs switch 语句（JDK 14+）

**A：**

### 核心区别

| 方面 | switch 语句 | switch 表达式 |
|------|------------|--------------|
| **语法** | `case X:` + `break;` | `case X ->` |
| **返回值** | 不能返回值 | 可以返回值 |
| **穿透** | 可穿透（fall-through） | 不穿透 |
| **返回值关键字** | - | `yield`（JDK 13+） |
| **可用位置** | 独立语句 | 可作为表达式返回值 |
| **JDK 版本** | JDK 1.0+ | JDK 14+（正式版） |

### switch 语句（旧语法）

```java
// switch 作为语句
int result;
switch (day) {
    case "MONDAY":
    case "FRIDAY":
        result = 6;
        break;
    case "TUESDAY":
        result = 7;
        break;
    default:
        result = 8;
}
System.out.println("工作时长: " + result + " 小时");
```

### switch 表达式（新语法）

```java
// switch 作为表达式，直接返回结果
int hours = switch (day) {
    case "MONDAY", "FRIDAY" -> 6;
    case "TUESDAY" -> 7;
    case "WEDNESDAY", "THURSDAY" -> 8;
    default -> {
        System.out.println("未知日期");
        yield 0;
    }
};
System.out.println("工作时长: " + hours + " 小时");
```

### yield 关键字（JDK 13+）

```java
// 为什么需要 yield？
// 因为箭头语法没有穿透，需要用 yield 返回值

String message = switch (status) {
    case PENDING -> "处理中";
    case APPROVED -> "已通过";
    case REJECTED -> {
        // 多行代码块，需要 yield 返回
        if (reason != null) {
            System.out.println("拒绝原因: " + reason);
        }
        yield "已拒绝";
    }
    default -> "未知状态";
};
```

### 对比示例

```java
// 场景：计算机星期几对应的中文

// 方式1：switch 语句
String getDayName(String day) {
    switch (day) {
        case "MONDAY": return "周一";
        case "TUESDAY": return "周二";
        case "WEDNESDAY": return "周三";
        case "THURSDAY": return "周四";
        case "FRIDAY": return "周五";
        case "SATURDAY": return "周六";
        case "SUNDAY": return "周日";
        default: return "未知";
    }
}

// 方式2：switch 表达式（JDK 14+）
String getDayName(String day) {
    return switch (day) {
        case "MONDAY" -> "周一";
        case "TUESDAY" -> "周二";
        case "WEDNESDAY" -> "周三";
        case "THURSDAY" -> "周四";
        case "FRIDAY" -> "周五";
        case "SATURDAY" -> "周六";
        case "SUNDAY" -> "周日";
        default -> "未知";
    };
}
```

### 编译器优化

```java
// switch 表达式要求穷尽所有情况（exhaustive）
// 编译器会检查是否覆盖所有 case

enum Color { RED, GREEN, BLUE }

// ❌ 编译错误：缺少 default
String name = switch (color) {
    case RED -> "红色";
    case GREEN -> "绿色";
    // BLUE 没有处理，编译错误
};

// ✅ 正确写法
String name = switch (color) {
    case RED -> "红色";
    case GREEN -> "绿色";
    case BLUE -> "蓝色";
};
```

---

## Q4: switch 的穿透（fall-through）特性如何正确使用？

**A：**

### 什么是穿透

```java
// 穿透示例
int n = 2;
switch (n) {
    case 1:
        System.out.println("1");    // 不执行
    case 2:
        System.out.println("2");    // 执行
    case 3:
        System.out.println("3");    // 执行（穿透！）
        break;                       // 遇到 break 才停止
    default:
        System.out.println("default");
}

// 输出：
// 2
// 3
```

### 穿透流程图

```
┌─────────────────────────────────────────────────────────────┐
│  switch (n) {                                               │
│                                                             │
│  case 1: ──→ 输出 "1" ──→ (无 break)                       │
│                  │                                          │
│                  ▼ 穿透                                     │
│  case 2: ──→ 输出 "2" ──→ (无 break)                       │
│                  │                                          │
│                  ▼ 穿透                                     │
│  case 3: ──→ 输出 "3" ──→ (有 break)                       │
│                  │                                          │
│                  ▼ 跳出 switch                              │
└─────────────────────────────────────────────────────────────┘
```

### 正确的穿透使用场景

```java
// 场景1：分组处理
switch (month) {
    case 1:
    case 2:
    case 3:
        System.out.println("第一季度");  // 1-3月都是第一季度
        break;
    case 4:
    case 5:
    case 6:
        System.out.println("第二季度");
        break;
    case 7:
    case 8:
    case 9:
        System.out.println("第三季度");
        break;
    case 10:
    case 11:
    case 12:
        System.out.println("第四季度");
        break;
}

// 场景2：共享初始逻辑
switch (command) {
    case "START":
    case "RESUME":
        initialize();  // 共同的初始化逻辑
        // 然后各自处理
        if (command.equals("START")) {
            start();
        } else {
            resume();
        }
        break;
    case "PAUSE":
        pause();
        break;
    case "STOP":
        stop();
        break;
}

// 场景3：实现多值匹配
switch (grade) {
    case 'A':
    case 'B':
    case 'C':
        System.out.println("及格");
        break;
    case 'D':
    case 'E':
    case 'F':
        System.out.println("不及格");
        break;
}
```

### 常见错误

```java
// ❌ 忘记 break 导致错误
switch (code) {
    case 200:
        System.out.println("成功");
    case 400:
        System.out.println("客户端错误");  // 200 也会执行到这里！
        break;
    case 500:
        System.out.println("服务器错误");
        break;
}

// ✅ 正确写法
switch (code) {
    case 200:
        System.out.println("成功");
        break;
    case 400:
        System.out.println("客户端错误");
        break;
    case 500:
        System.out.println("服务器错误");
        break;
}
```

### switch 表达式中的穿透

```java
// switch 表达式不支持穿透（箭头语法）
// 但可以用逗号分隔多个值

String result = switch (day) {
    case "MONDAY", "FRIDAY" -> "工作日";
    case "SATURDAY", "SUNDAY" -> "周末";
    default -> "未知";
};
```

---

## Q5: switch 的穿透在枚举场景下的注意事项？

**A：**

### 枚举与 switch 的结合

```java
enum Status {
    PENDING, APPROVED, REJECTED, CANCELLED
}

// 使用 switch 处理枚举
Status status = Status.APPROVED;
switch (status) {
    case PENDING:
        System.out.println("待处理");
        break;
    case APPROVED:
        System.out.println("已批准");
        break;
    case REJECTED:
        System.out.println("已拒绝");
        break;
    case CANCELLED:
        System.out.println("已取消");
        break;
}
```

### 穿透在枚举中的正确使用

```java
// 按状态类型分组
switch (status) {
    case PENDING:
    case CANCELLED:
        System.out.println("未完成状态");
        break;
    case APPROVED:
    case REJECTED:
        System.out.println("已完成状态");
        break;
}
```

### 注意事项

```java
// ⚠️ 注意1：case 后面直接写枚举值，不需要类型前缀
switch (status) {
    case Status.PENDING:  // ❌ 错误写法
        break;
    case PENDING:         // ✅ 正确写法
        break;
}

// ⚠️ 注意2：switch 表达式（JDK 14+）更简洁
String message = switch (status) {
    case PENDING -> "待处理";
    case APPROVED -> "已批准";
    case REJECTED -> "已拒绝";
    case CANCELLED -> "已取消";
};

// ⚠️ 注意3：遗漏枚举值会编译错误（JDK 14+ switch 表达式）
// 如果新增枚举值，旧代码的 switch 会提示不穷尽
enum Status {
    PENDING, APPROVED, REJECTED, CANCELLED, SUSPENDED  // 新增 SUSPENDED
}

// 旧 switch 表达式会编译错误
String message = switch (status) {
    case PENDING -> "待处理";
    case APPROVED -> "已批准";
    // 缺少 REJECTED, CANCELLED, SUSPENDED
    // 编译错误：枚举常量被覆盖
};
```

### 最佳实践

```java
// 推荐：使用 switch 表达式 + 枚举
enum Level {
    LOW, MEDIUM, HIGH, CRITICAL
}

String getAlert(Level level) {
    return switch (level) {
        case LOW -> "低优先级";
        case MEDIUM -> "中等优先级";
        case HIGH -> "高优先级";
        case CRITICAL -> "严重";
    };
}

// 如果需要添加新枚举值，编译器会强制更新所有 switch
```

---

## Q6: 用 switch 改写多层 if-else 的例子？

**A：**

### 多层 if-else 示例

```java
// 原代码：多层 if-else
public String getGrade(int score) {
    if (score >= 90) {
        return "A";
    } else if (score >= 80) {
        return "B";
    } else if (score >= 70) {
        return "C";
    } else if (score >= 60) {
        return "D";
    } else {
        return "F";
    }
}
```

### 改写为 switch

```java
// 方法1：使用范围计算 + switch
public String getGrade(int score) {
    if (score < 0 || score > 100) {
        return "无效分数";
    }
    
    return switch (score / 10) {
        case 10, 9 -> "A";  // 90-100
        case 8 -> "B";       // 80-89
        case 7 -> "C";       // 70-79
        case 6 -> "D";       // 60-69
        default -> "F";      // 0-59
    };
}
```

### 更多改写示例

```java
// 场景1：状态机
// if-else 版本
public String handleOrder(String status) {
    if ("CREATED".equals(status)) {
        return "订单已创建";
    } else if ("PAID".equals(status)) {
        return "订单已支付";
    } else if ("SHIPPED".equals(status)) {
        return "订单已发货";
    } else if ("DELIVERED".equals(status)) {
        return "订单已送达";
    } else if ("CANCELLED".equals(status)) {
        return "订单已取消";
    } else {
        return "未知状态";
    }
}

// switch 版本（JDK 14+）
public String handleOrder(String status) {
    return switch (status) {
        case "CREATED" -> "订单已创建";
        case "PAID" -> "订单已支付";
        case "SHIPPED" -> "订单已发货";
        case "DELIVERED" -> "订单已送达";
        case "CANCELLED" -> "订单已取消";
        default -> "未知状态";
    };
}

// 场景2：计算器
// if-else 版本
public double calculate(char op, double a, double b) {
    if (op == '+') {
        return a + b;
    } else if (op == '-') {
        return a - b;
    } else if (op == '*') {
        return a * b;
    } else if (op == '/') {
        if (b != 0) {
            return a / b;
        }
    }
    return 0;
}

// switch 版本（JDK 14+）
public double calculate(char op, double a, double b) {
    return switch (op) {
        case '+' -> a + b;
        case '-' -> a - b;
        case '*' -> a * b;
        case '/' -> b != 0 ? a / b : 0;
        default -> 0;
    };
}
```

### 改写原则

```
改写条件：
├─ 条件是等值判断（不是范围）
├─ 判断的值类型支持 switch
└─ 分支数量适中（3-10个）

if-else 更适合的场景：
├─ 范围判断（age >= 18）
├─ 复合条件（age >= 18 && hasLicense）
└─ 布尔表达式
```

---

## Q7: switch 多标签语法（case A, B, C:）的用法（JDK 14+）？

**A：**

### 基本语法

```java
// 多个 case 共享同一代码块
switch (day) {
    case "MONDAY", "FRIDAY", "SUNDAY":
        System.out.println("特殊日期");
        break;
    case "TUESDAY":
        System.out.println("普通工作日");
        break;
    default:
        System.out.println("其他日期");
}
```

### 与箭头语法的结合

```java
// JDK 14+ 箭头语法 + 多标签
String result = switch (month) {
    case 1, 2, 3 -> "第一季度";
    case 4, 5, 6 -> "第二季度";
    case 7, 8, 9 -> "第三季度";
    case 10, 11, 12 -> "第四季度";
    default -> "无效月份";
};

// 多标签 + yield
String getCategory(int code) {
    return switch (code) {
        case 100, 101, 102 -> {
            System.out.println("信息类消息");
            yield "INFO";
        }
        case 200, 201, 204 -> {
            System.out.println("成功类消息");
            yield "SUCCESS";
        }
        case 400, 401, 403, 404 -> {
            System.out.println("客户端错误");
            yield "CLIENT_ERROR";
        }
        case 500, 502, 503 -> {
            System.out.println("服务器错误");
            yield "SERVER_ERROR";
        }
        default -> "UNKNOWN";
    };
}
```

### 枚举多标签

```java
enum Size { SMALL, MEDIUM, LARGE, EXTRA_LARGE }

String getLabel(Size size) {
    return switch (size) {
        case SMALL, MEDIUM -> "常规尺码";
        case LARGE, EXTRA_LARGE -> "大码";
    };
}
```

### 与传统穿透语法的对比

```java
// 传统穿透写法
switch (month) {
    case 1:
    case 2:
    case 3:
        System.out.println("第一季度");
        break;
    // ...
}

// 多标签写法（更简洁）
switch (month) {
    case 1, 2, 3:
        System.out.println("第一季度");
        break;
    // ...
}
```

### 常见用法

```java
// 1. HTTP 状态码分组
int statusCode = 404;
String category = switch (statusCode) {
    case 200, 201, 204, 206 -> "成功";
    case 301, 302, 304, 307, 308 -> "重定向";
    case 400, 401, 403, 404, 405, 408, 429 -> "客户端错误";
    case 500, 501, 502, 503, 504 -> "服务器错误";
    default -> "其他";
};

// 2. 字符分类
char c = 'e';
String type = switch (c) {
    case 'a', 'e', 'i', 'o', 'u' -> "元音字母";
    case 'b', 'c', 'd', 'f', 'g', 'h', 'j', 'k', 'l', 'm',
         'n', 'p', 'q', 'r', 's', 't', 'v', 'w', 'x', 'y', 'z' -> "辅音字母";
    case '0', '1', '2', '3', '4', '5', '6', '7', '8', '9' -> "数字";
    default -> "其他字符";
};

// 3. 布尔值组合
boolean isWeekend = true;
boolean isHoliday = false;
String dayType = switch ((isWeekend ? 1 : 0) + (isHoliday ? 2 : 0)) {
    case 0 -> "工作日";
    case 1 -> "普通周末";
    case 2 -> "工作日但放假";
    case 3 -> "周末且放假";
    default -> throw new IllegalStateException();
};
```

### 注意事项

```java
// ⚠️ case 值必须是编译时常量
final int A = 1;
final int B = 2;
switch (x) {
    case A, B -> "A 或 B";  // ✅ 可以
    case 1, 2 -> "1 或 2";  // ✅ 可以
}

// ⚠️ case 值不能重复
switch (x) {
    case 1, 1 -> "重复";    // ❌ 编译错误
    case 2 -> "OK";
}
```
