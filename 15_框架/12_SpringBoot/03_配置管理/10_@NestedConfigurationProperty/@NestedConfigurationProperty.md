# @NestedConfigurationProperty

## 先说结论

`@NestedConfigurationProperty` 用于标记嵌套配置属性，让 IDE 在 application.yml 中提供准确的属性提示，避免扁平化展示。

## 深度解析

### 问题背景

不加 `@NestedConfigurationProperty` 时，IDE 可能这样显示：

```
server.port          ✓
server.servlet       ✗ (扁平展示)
server.servlet.context-path  ✓
```

### 基本用法

```java
@ConfigurationProperties(prefix = "server")
public class ServerProperties {

    private int port = 8080;

    @NestedConfigurationProperty
    private Servlet servlet = new Servlet();

    @NestedConfigurationProperty
    private Error error = new Error();

    // getter/setter...
}
```

```java
public static class Servlet {
    private String contextPath = "/";
}

public static class Error {
    private String path = "/error";
}
```

### 效果对比

| 不加注解 | 加注解 |
|----------|--------|
| IDE 提示扁平化 | IDE 提示层级化 |
| `server.servlet` 显示为简单属性 | `server.servlet.contextPath` 正确嵌套 |

### YAML 配置

```yaml
server:
  port: 8080
  servlet:
    contextPath: /api
  error:
    path: /custom-error
```

## 易错点/踩坑

- ❌ 嵌套类是 public static — 推荐，否则无法访问
- ❌ 忘记加 getter/setter — 配置绑定失败
- ❌ 嵌套对象未初始化 — 可能 NPE

## 关联知识点

- [[07_@ConfigurationProperties配置绑定]] — 配置绑定基础
- [[27_配置元数据]] — IDE 提示机制
