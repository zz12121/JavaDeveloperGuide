# @ConfigurationProperties vs @Value

## 先说结论

| 特性 | @ConfigurationProperties | @Value |
|------|------------------------|--------|
| 适用场景 | 结构化配置、批量绑定 | 单个简单值 |
| 类型安全 | ✅ | ❌ |
| 松散绑定 | ✅ | ❌ |
| SpEL 支持 | ❌ | ✅ |
| 数据校验 | ✅ | ❌ |
| IDE 提示 | ✅ | ❌ |

## 深度解析

### @ConfigurationProperties 优势

```java
// 批量绑定，代码简洁
@ConfigurationProperties(prefix = "server")
public class ServerProperties {
    private int port;
    private String contextPath;
    private Error error = new Error();

    // getter/setter...
}

// application.yml
server:
  port: 8080
  context-path: /api
  error:
    path: /error
```

### @Value 优势

```java
// 简单值读取
@Value("${server.port:8080}")
private int port;

// SpEL 表达式
@Value("#{systemProperties['user.dir']}")
private String userDir;

// 复杂默认值
@Value("${app.timeout:#{30 * 1000}}")
private int timeout;
```

### 对比表

| 维度 | @ConfigurationProperties | @Value |
|------|------------------------|--------|
| 配置项数量 | 多个相关配置 | 单个配置 |
| 代码风格 | 配置类 + 属性 | 直接注入 |
| 层级结构 | 原生支持嵌套 | 需要手动解析 |
| 默认值 | 属性定义默认值 | `:default` 语法 |
| 转换器 | 自动应用 | 自动应用 |
| 校验 | @Validated | ❌ |
| 刷新 | @RefreshScope 支持 | @RefreshScope 支持 |

### 选择建议

```
场景：
  ├── 单个简单值 → @Value
  ├── 多个相关配置 → @ConfigurationProperties
  ├── 需要校验 → @ConfigurationProperties + @Validated
  ├── 需要 SpEL → @Value
  └── 配置类（第三方） → @ConfigurationProperties
```

## 易错点/踩坑

- ❌ 所有配置都用 @Value — 配置项多时代码混乱
- ❌ 所有配置都用 @ConfigurationProperties — 简单值没必要
- ❌ 混淆两者场景 — 按需选择

## 关联知识点

- [[05_@Value注解读取配置]] — @Value 详解
- [[07_@ConfigurationProperties配置绑定]] — 配置绑定详解
- [[26_Relaxed属性名绑定]] — 松散绑定规则
