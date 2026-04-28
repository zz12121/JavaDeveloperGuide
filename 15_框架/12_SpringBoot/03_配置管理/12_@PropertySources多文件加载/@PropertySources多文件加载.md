# @PropertySources 多文件加载

## 先说结论

`@PropertySources` 是 `@PropertySource` 的复数形式，用于同时加载多个配置文件，是 `@PropertySource` 的容器注解。

## 深度解析

### 基本用法

```java
@Configuration
@PropertySources({
    @PropertySource("classpath:db.properties"),
    @PropertySource("classpath:redis.properties"),
    @PropertySource(value = "file:./custom.properties", ignoreResourceNotFound = true)
})
public class AppConfig { }
```

### 等价写法

```java
// 方式一：@PropertySources（多个）
@PropertySources({
    @PropertySource("classpath:db.properties"),
    @PropertySource("classpath:app.properties")
})

// 方式二：多个 @PropertySource（Java 8+ 可重复注解）
@PropertySource("classpath:db.properties")
@PropertySource("classpath:app.properties")
```

### Spring Boot 2.4+ 推荐方式

**推荐用 `spring.config.import`**：

```yaml
# application.yml
spring:
  config:
    import: optional:file:./db.properties, optional:file:./redis.properties
```

```java
// 不再用 @PropertySource
// 直接在 application.yml 中导入
```

## 易错点/踩坑

- ❌ 路径错误导致启动失败 — 加 `ignoreResourceNotFound = true`
- ❌ 同名属性被覆盖 — 注意加载顺序
- ❌ 混用 @PropertySource 和 spring.config.import — 可能导致混乱

## 关联知识点

- [[11_@PropertySource加载外部配置]] — 单文件加载
- [[24_spring_config_import导入配置]] — 2.4+ 推荐方式
- [[14_SpringBoot配置优先级]] — 加载顺序
