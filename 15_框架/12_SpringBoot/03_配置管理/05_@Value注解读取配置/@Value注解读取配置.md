# @Value 注解读取配置

## 先说结论

`@Value` 是最基础的配置读取方式，适合单个配置项的场景，比 `@ConfigurationProperties` 更轻量但功能较弱。

## 深度解析

### 基本语法

```java
@Value("${server.port}")
private int port;

@Value("${server.servlet.context-path}")
private String contextPath;
```

### 支持的类型

```java
@Value("${app.name}")           // String
@Value("${app.port:8080}")      // int（带默认值）
@Value("${app.enabled:true}")   // boolean
@Value("${app.rate:0.5}")       // double
@Value("${app.list:1,2,3}")     // List<String>
```

### SpEL 表达式

```java
// 简单计算
@Value("#{2 * 3}")
private int result;  // 6

// 调用方法
@Value("#{'${app.name}'.toUpperCase()}")
private String name;

// 复杂表达式
@Value("#{systemProperties['user.dir']}")
private String userDir;
```

### 数组和集合

```java
// 逗号分隔
@Value("${app.servers:127.0.0.1,192.168.1.1}")
private List<String> servers;

// JSON 数组
@Value("#{'${app.array}'.split(',')}")
private String[] array;
```

## 易错点/踩坑

- ❌ `@Value` 注入 static 字段 — 不生效，必须是实例字段
- ❌ 缺少默认值且配置不存在 — 启动报错
- ❌ 用 `@Value` 读取大量配置 — 应该用 `@ConfigurationProperties`
- ❌ 循环依赖时使用 — 可能导致初始化顺序问题

## 代码示例

### 完整示例

```java
@Component
public class AppConfig {

    @Value("${server.port:8080}")
    private int port;

    @Value("${spring.application.name:unknown}")
    private String appName;

    @Value("${app.features.cache:true}")
    private boolean cacheEnabled;

    @Value("${app.retry.max:3}")
    private int maxRetries;

    public void printConfig() {
        System.out.println("Port: " + port);
        System.out.println("App: " + appName);
        System.out.println("Cache: " + cacheEnabled);
        System.out.println("Max Retries: " + maxRetries);
    }
}
```

### YAML 配置

```yaml
server:
  port: 8080
spring:
  application:
    name: myapp
app:
  features:
    cache: true
  retry:
    max: 3
```

## @Value vs @ConfigurationProperties

| 特性 | @Value | @ConfigurationProperties |
|------|--------|-------------------------|
| 批量绑定 | ❌ | ✅ |
| 类型安全 | ❌ | ✅ |
| 松散绑定 | ❌ | ✅ |
| SpEL 支持 | ✅ | ❌ |
| 校验 | ❌ | ✅ (@Validated) |
| 默认值 | 需要 `:default` | 可以在 POJO 定义 |

## 关联知识点

- [[09_@ConfigurationProperties_vs_@Value]] — 详细对比
- [[06_@Value_${}_vs_#{}区别]] — 占位符 vs SpEL
- [[07_@ConfigurationProperties配置绑定]] — 批量绑定
