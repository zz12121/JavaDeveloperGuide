# 自定义 Starter 自动配置

## Q1：Spring Boot Starter 的标准命名是什么？

**A**：

| 类型 | 命名规范 | 示例 |
|------|----------|------|
| 官方 Starter | `spring-boot-starter-xxx` | `spring-boot-starter-web` |
| 第三方 Starter | `xxx-spring-boot-starter` | `druid-spring-boot-starter` |
| 自动配置模块 | `xxx-spring-boot-autoconfigure` | `mybatis-spring-boot-autoconfigure` |

---

## Q2：如何创建完整的自定义 Starter？

**A**：分为两步：

**1. 创建自动配置模块**

```java
// XxxAutoConfiguration.java
@Configuration
@ConditionalOnClass(XxxService.class)
@EnableConfigurationProperties(XxxProperties.class)
public class XxxAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public XxxService xxxService(XxxProperties properties) {
        return new XxxServiceImpl(properties);
    }
}
```

```text
# META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
com.example.xxx.XxxAutoConfiguration
```

**2. 创建 Starter 模块**

```xml
<dependencies>
    <dependency>
        <groupId>com.example</groupId>
        <artifactId>xxx-spring-boot-autoconfigure</artifactId>
        <version>1.0.0</version>
    </dependency>
    <!-- 用户使用 Starter 时需要提供的依赖 -->
</dependencies>
```

用户只需引入 Starter：
```xml
<dependency>
    <groupId>com.example</groupId>
    <artifactId>xxx-spring-boot-starter</artifactId>
</dependency>
```

---

## Q3：为什么 Starter 的自动配置类要放在单独的模块？

**A**：

| 模块 | 作用 |
|------|------|
| `autoconfigure` | 包含自动配置类，只依赖核心库 |
| `starter` | 聚合 autoconfigure + 用户依赖 |

**好处**：
- 用户只需引入一个 Starter 依赖
- Starter 可以控制可选依赖的引入方式
- 避免循环依赖

---

## Q4：如何让 Starter 可配置？

**A**：使用 `@ConfigurationProperties`：

```java
@ConfigurationProperties(prefix = "app.xxx")
public class XxxProperties {

    private boolean enabled = true;
    private String url;
    private int timeout = 3000;

    // getters and setters
}

@Configuration
@EnableConfigurationProperties(XxxProperties.class)
public class XxxAutoConfiguration {

    @Autowired
    private XxxProperties properties;

    @Bean
    @ConditionalOnMissingBean
    @ConditionalOnProperty(name = "app.xxx.enabled", havingValue = "true", matchIfMissing = true)
    public XxxService xxxService() {
        return new XxxServiceImpl(properties);
    }
}
```

用户配置：
```yaml
app:
  xxx:
    enabled: true
    url: http://api.example.com
    timeout: 5000
```

---

## Q5：如何避免 Starter 和自动配置类的依赖冲突？

**A**：

```xml
<!-- Starter 中标记可选依赖 -->
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid</artifactId>
    <optional>true</optional>
</dependency>
```

使用 `@ConditionalOnClass(name = "...")` 避免编译时类不存在：

```java
@ConditionalOnClass(name = "com.alibaba.druid.DruidDataSource")
// 运行时才检查，编译时不依赖
```

