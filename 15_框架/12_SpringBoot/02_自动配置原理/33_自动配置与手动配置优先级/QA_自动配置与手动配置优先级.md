# 自动配置与手动配置优先级

## Q1：为什么自动配置不会覆盖用户的配置？

**A**：通过 `@ConditionalOnMissingBean` 实现：

```java
@AutoConfiguration
public class DataSourceAutoConfiguration {

    @Bean
    @Primary
    @ConditionalOnMissingBean        // 关键：只有不存在时才注册
    public DataSource dataSource() {
        return new HikariDataSource();
    }
}
```

```java
@Configuration
public class MyConfig {
    @Bean
    public DataSource dataSource() {  // 用户先注册
        return new DruidDataSource();
    }
}
```

执行顺序：
1. `@ComponentScan` 扫描 → `MyConfig.dataSource()` 注册
2. 自动配置加载 → `@ConditionalOnMissingBean` 检测到已存在 → **跳过**

---

## Q2：用户 Bean 和自动配置 Bean 重名怎么办？

**A**：会抛 `BeanDefinitionOverrideException`：

```
beans.BootstrapException: Failed to instantiate [org.springframework.boot.jdbc.DataSourceBuilder]:
Caused by: org.springframework.beans.factory.BeanDefinitionOverrideException:
  Invalid bean definition with name 'dataSource'. 
  Only one bean of each name may be defined.
```

**解决**：
1. 使用 `@Primary` 标记优先级
2. 避免重名
3. 排除自动配置

---

## Q3：@Primary 在自动配置和手动配置中同时出现会怎样？

**A**：会报错：

```java
@AutoConfiguration
public class AutoConfig {
    @Bean
    @Primary
    public MyService serviceA() { return new ServiceA(); }
}

@Configuration
public class UserConfig {
    @Bean
    @Primary
    public MyService serviceB() { return new ServiceB(); }
}
```

```
No qualifying bean of type 'MyService' available: 
  more than one 'primary' bean found among candidates
```

**解决**：只保留一个 `@Primary`，或在注入时明确指定：
```java
@Autowired
@Qualifier("serviceB")
private MyService myService;
```

---

## Q4：如何让自动配置 Bean 覆盖用户配置？

**A**：

```properties
# application.yml
spring.main.allow-bean-definition-overriding=true
```

**注意**：这是危险设置，可能导致意外行为。通常应该：
1. 用户配置优先
2. 自动配置作为兜底

---

## Q5：多个自动配置之间的优先级如何决定？

**A**：

| 方式 | 说明 |
|------|------|
| `@AutoConfigureOrder` | 显式指定顺序值 |
| `@AutoConfigureBefore/After` | 相对顺序 |
| 类名字母顺序 | 未指定时，按字母顺序 |

```java
// AAuto 在 BAuto 之前加载
@AutoConfiguration
@AutoConfigureBefore(BAutoConfiguration.class)
public class AAutoConfiguration { }
```

