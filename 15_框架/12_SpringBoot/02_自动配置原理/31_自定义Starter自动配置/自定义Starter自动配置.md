# 自定义 Starter 自动配置

## 先说结论

Spring Boot Starter 是一组便捷的依赖描述符，包含自动配置类和相关支持库。通过自定义 Starter，可以封装通用功能模块，实现"开箱即用"的同时保持灵活配置。

## 深度解析

### Starter 命名规范

```
# 官方 starter
spring-boot-starter-xxx

# 第三方 starter
xxx-spring-boot-starter
```

| 类型 | 命名 | 说明 |
|------|------|------|
| 官方 | `spring-boot-starter-xxx` | Spring Boot 官方提供 |
| 自动配置 | `xxx-spring-boot-autoconfigure` | 自动配置模块 |
| Starter | `xxx-spring-boot-starter` | 用户依赖的 Starter |
| 两合一 | `xxx-spring-boot-starter` | 同时包含自动配置 |

### 完整 Starter 结构

```
xxx-spring-boot-starter/
├── pom.xml
├── src/main/resources/
│   └── META-INF/
│       └── spring/
│           └── org.springframework.boot.autoconfigure.AutoConfiguration.imports
└── src/main/java/
    └── com/example/
        └── xxx/
            ├── XxxAutoConfiguration.java
            └── XxxProperties.java
```

### 自动配置类模板

```java
@Configuration
@ConditionalOnClass(XxxService.class)           // 依赖检查
@EnableConfigurationProperties(XxxProperties.class)  // 配置属性绑定
@AutoConfigureAfter(DataSourceAutoConfiguration.class)  // 排序
public class XxxAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public XxxService xxxService(XxxProperties props) {
        return new XxxServiceImpl(props);
    }
}
```

### 属性配置类

```java
@ConfigurationProperties(prefix = "my.xxx")
public class XxxProperties {

    private boolean enabled = true;
    private String host = "localhost";
    private int port = 8080;

    // getters and setters
}
```

## 易错点/踩坑

- ❌ **命名不符合规范** — 用户可能找不到你的 Starter
- ❌ **缺少条件注解** — 没有 classpath 检查会导致无依赖时编译失败
- ❌ **使用 Class 引用而非 String** — 可选依赖应该用 String 类名

## 关联知识点

- [[SpringBoot2_7自动配置变化]] — 新格式支持
- [[AutoConfiguration_imports文件]] — 配置声明
