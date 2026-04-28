# Relaxed 属性名绑定

## 先说结论

Spring Boot 的 Relaxed 绑定允许使用不同格式的属性名绑定到同一配置，驼峰、横线、下划线自动兼容。

## 深度解析

### 绑定示例

| 配置属性 | 等价写法 |
|----------|----------|
| `server.port` | `server.port`, `serverPort`, `SERVER_PORT`, `SERVER_PORT` |
| `server.servlet.context-path` | `server.servlet.contextPath`, `SERVER_SERVLET_CONTEXT_PATH` |

### 绑定规则

```
server.servlet.context-path
         ↓
允许的变体：
- server.servlet.context-path
- server.servlet.contextPath
- server.servlet.context_path
- SERVER.SERVLET.CONTEXT-PATH
- SERVER_SERVLET_CONTEXT_PATH
```

### 实际应用

```yaml
# application.yml
server:
  servlet:
    context-path: /api
```

```java
// 都能读取
@Value("${server.servlet.context-path}")    // ✓
@Value("${server.servlet.contextPath}")    // ✓
@Value("${SERVER_SERVLET_CONTEXT_PATH}")   // ✓
```

### @ConfigurationProperties

```java
@ConfigurationProperties(prefix = "app-config")
public class AppProperties {
    private String userName;  // 可以用 app-config.user-name 赋值
}
```

### 环境变量特殊规则

| 配置属性 | 环境变量 |
|----------|----------|
| `app.config-file` | `APP_CONFIG_FILE` 或 `APPCONFIGFILE` |
| `app.configFile` | `APP_CONFIGFILE` |

## 易错点/踩坑

- ❌ 大小写混淆 — `userName` 和 `username` 是不同属性
- ❌ 前缀不匹配 — `@ConfigurationProperties(prefix = "app")` 不匹配 `app-config.name`
- ❌ IDE 提示不准确 — 部分变体可能不显示

## 关联知识点

- [[07_@ConfigurationProperties配置绑定]] — 配置绑定
- [[17_环境变量自动映射]] — 环境变量转换
