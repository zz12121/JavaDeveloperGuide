# @ConfigurationProperties 配置绑定

## 先说结论

`@ConfigurationProperties` 是 Spring Boot 推荐的结构化配置方式，将配置文件中的属性批量绑定到 POJO 类，类型安全、层级清晰。

## 深度解析

### 基本用法

```java
// 配置类
@Component
@ConfigurationProperties(prefix = "server")
public class ServerProperties {
    private int port;
    private String contextPath;

    // getter/setter
}

// application.yml
server:
  port: 8080
  context-path: /api
```

### 与 @Component 配合

```java
@Component
@ConfigurationProperties(prefix = "app")
@Data  // Lombok
public class AppProperties {
    private String name;
    private int maxConnections;
    private List<String> allowedOrigins;
}
```

### 启用配置属性

有两种方式启用 `@ConfigurationProperties`：

```java
// 方式一：在配置类上加 @EnableConfigurationProperties
@SpringBootApplication
@EnableConfigurationProperties(AppProperties.class)
public class Application { }

// 方式二：配置类自身加 @ConfigurationProperties + @Component
@Configuration
@ConfigurationProperties(prefix = "app")
public class AppProperties { }
```

### 嵌套绑定

```java
@ConfigurationProperties(prefix = "server")
public class ServerProperties {
    private int port;
    private Servlet servlet = new Servlet();
    private Error error = new Error();

    @Data
    public static class Servlet {
        private String contextPath = "/";
    }

    @Data
    public static class Error {
        private String path = "/error";
    }
}
```

对应 YAML：
```yaml
server:
  port: 8080
  servlet:
    context-path: /app
  error:
    path: /custom-error
```

## 易错点/踩坑

- ❌ 忘记加 getter/setter — 绑定需要， Lombok 可简化
- ❌ prefix 写错 — 严格匹配，区分大小写
- ❌ 配置类未注册为 Bean — 需要 `@Component` 或 `@EnableConfigurationProperties`
- ❌ 初始化循环依赖 — 绑定时可能未初始化完成

## 代码示例

```java
// 完整示例
@Data
@Component
@ConfigurationProperties(prefix = "spring.datasource")
public class DataSourceProperties {
    private String url;
    private String username;
    private String password;
    private int maxPoolSize = 10;
    private int minIdle = 5;
}

// 使用
@RestController
public class TestController {
    @Autowired
    private DataSourceProperties dsProps;

    @GetMapping("/info")
    public String info() {
        return "Pool: " + dsProps.getMaxPoolSize();
    }
}
```

## 关联知识点

- [[08_@EnableConfigurationProperties]] — 启用配置绑定
- [[09_@ConfigurationProperties_vs_@Value]] — 对比选择
- [[10_@NestedConfigurationProperty]] — 嵌套属性标记
