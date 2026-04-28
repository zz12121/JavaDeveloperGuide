# @Value ${} vs #{} 区别

## 先说结论

| 语法 | 用途 | 示例 |
|------|------|------|
| `${}` | 属性占位符，读取配置值 | `${server.port}` |
| `#{}` | SpEL 表达式，执行运算 | `#{2 * 3}` |

## 深度解析

### ${} 属性占位符

```java
// 读取配置文件的值
@Value("${server.port}")
private int port;

// 带默认值
@Value("${app.timeout:5000}")
private int timeout;

// 读取系统属性
@Value("${user.dir}")
private String userDir;
```

### #{} SpEL 表达式

```java
// 数学运算
@Value("#{2 + 3}")
private int sum;  // 5

// 方法调用
@Value("#{'${app.name}'.toUpperCase()}")
private String name;

// 对象属性
@Value("#{systemProperties['user.name']}")
private String user;

// Bean 方法
@Value("#{@dataSource.url}")
private String url;

// 条件表达式
@Value("#{2 > 1 ? 'yes' : 'no'}")
private String result;
```

### 混合使用

```java
// #{} 中嵌套 ${}
@Value("#{'${app.db.type}'.toUpperCase()}")
private String dbType;

// ${} 中嵌套 #{}（不常见）
@Value("${app.max:#{100}}")
private int max;
```

### 对比表

| 特性 | `${}` | `#{}` |
|------|-------|-------|
| 功能 | 读取配置值 | 执行表达式 |
| 支持计算 | ❌ | ✅ |
| 调用方法 | ❌ | ✅ |
| 访问 Bean | ❌ | ✅ |
| 默认值 | `:default` | `#defaultValue` |
| 复杂度 | 简单 | 复杂 |

## 易错点/踩坑

- ❌ `#${server.port}` — 顺序错了
- ❌ `${#bean.method}` — `${}` 不支持 SpEL
- ❌ 混淆两者用途 — `${}` 读配置，`#{}` 做运算

## 关联知识点

- [[05_@Value注解读取配置]] — @Value 基础
- [[25_配置占位符]] — 占位符详解
