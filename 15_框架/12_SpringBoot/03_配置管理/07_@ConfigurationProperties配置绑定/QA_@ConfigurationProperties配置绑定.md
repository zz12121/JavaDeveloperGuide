# @ConfigurationProperties 配置绑定 QA

## Q1：@ConfigurationProperties 和 @Value 哪个更好？

**A**：根据场景选择：

| 场景 | 推荐 |
|------|------|
| 少量配置项 | @Value |
| 大量相关配置（配置类） | @ConfigurationProperties |
| 需要校验 | @ConfigurationProperties + @Validated |
| 需要 IDE 提示 | @ConfigurationProperties（配合 processor） |

---

## Q2：一定要加 getter/setter 吗？

**A**：是的，Spring Boot 通过 setter 绑定值。如果不想写，可以用 Lombok：

```java
@Data
@Component
@ConfigurationProperties(prefix = "app")
public class AppProperties { }
```

---

## Q3：prefix 怎么写才对？

**A**：prefix 严格匹配配置文件的键：

```java
@ConfigurationProperties(prefix = "server.servlet")
// 对应
server:
  servlet:
    context-path: /api
```

---

## Q4：可以用 @ConfigurationProperties 绑定第三方库的配置吗？

**A**：可以，Spring Boot 的自动配置类就是用这个方式：

```java
@EnableConfigurationProperties(DataSourceProperties.class)
public class DataSourceAutoConfiguration { }
```

---

## Q5：配置类加载顺序有问题怎么办？

**A**：
```java
// 使用 @DependsOn 或调整 Bean 定义顺序
@Bean
@ConfigurationProperties(prefix = "app")
@DependsOn("otherBean")
public AppProperties appProperties() {
    return new AppProperties();
}
```

---

## Q6：可以不用 @Component 吗？

**A**：可以，用 `@EnableConfigurationProperties` 注册：

```java
@SpringBootApplication
@EnableConfigurationProperties(AppProperties.class)
public class App { }

// AppProperties 不需要 @Component
@ConfigurationProperties(prefix = "app")
public class AppProperties { }
```
