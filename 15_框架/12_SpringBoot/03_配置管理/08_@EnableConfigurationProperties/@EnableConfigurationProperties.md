# @EnableConfigurationProperties

## 先说结论

`@EnableConfigurationProperties` 是启用 `@ConfigurationProperties` 类的标准方式，通常配合自动配置使用。

## 深度解析

### 基本用法

```java
@SpringBootApplication
@EnableConfigurationProperties(AppProperties.class)
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}

// 配置类（不需要 @Component）
@ConfigurationProperties(prefix = "app")
public class AppProperties {
    private String name;
    private int maxSize;
}
```

### 在自动配置类中使用

```java
// MyStarterAutoConfiguration.java
@Configuration
@EnableConfigurationProperties(MyProperties.class)
public class MyStarterAutoConfiguration {

    @Autowired
    private MyProperties properties;

    @Bean
    public MyService myService() {
        return new MyService(properties);
    }
}

// MyProperties.java
@ConfigurationProperties(prefix = "my")
public class MyProperties {
    private String option = "default";
}
```

### 与 @Component 对比

| 特性 | @EnableConfigurationProperties | @Component |
|------|--------------------------------|------------|
| 放置位置 | 配置类/启动类 | 配置属性类本身 |
| 配置属性类 | 不需要 @Component | 需要 @Component |
| 适用场景 | 自动配置、第三方库 | 自定义配置 |
| @Validated 支持 | ✅ | ✅ |

### 配合 @Validated 使用

```java
@SpringBootApplication
@EnableConfigurationProperties(AppProperties.class)
@Validated
public class Application { }

@ConfigurationProperties(prefix = "app")
@Validated
public class AppProperties {
    @NotBlank
    private String name;

    @Min(1)
    @Max(100)
    private int maxSize;
}
```

## 易错点/踩坑

- ❌ 在多个地方重复启用 — 只需启用一次
- ❌ 配置类忘记加 getter/setter — 绑定失败
- ❌ 同时加 @Component 和 @EnableConfigurationProperties — 重复注册

## 关联知识点

- [[07_@ConfigurationProperties配置绑定]] — 配置绑定基础
- [[09_@ConfigurationProperties_vs_@Value]] — 对比选择
- [[27_配置元数据]] — IDE 提示支持
