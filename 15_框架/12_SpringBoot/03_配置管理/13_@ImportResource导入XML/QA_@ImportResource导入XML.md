# @ImportResource 导入 XML QA

## Q1：@ImportResource 和 @ComponentScan 有什么区别？

**A**：

| 特性 | @ImportResource | @ComponentScan |
|------|----------------|----------------|
| 导入内容 | XML 配置文件 | 注解扫描包 |
| 位置 | classpath/文件路径 | 包名 |
| 适用场景 | 遗留 XML | 注解配置类 |

---

## Q2：XML 中的占位符 `${}` 能解析 application.yml 吗？

**A**：需要配置：

```java
@Configuration
@ImportResource("classpath:beans.xml")
public class AppConfig {
    @Bean
    public static PropertySourcesPlaceholderConfigurer propertyConfigurer() {
        return new PropertySourcesPlaceholderConfigurer();
    }
}
```

---

## Q3：XML 导入的 Bean 和注解的 Bean 冲突怎么办？

**A**：后加载的覆盖先加载的：
- `@ComponentScan` 先扫描
- `@ImportResource` 后导入

所以 XML 中的同名 Bean 会覆盖注解的。

---

## Q4：现代 Spring Boot 项目还需要 @ImportResource 吗？

**A**：一般不需要。Spring Boot 推崇全注解配置，推荐：
- XML → Java @Configuration
- `<bean>` → @Bean
- `<property>` → @Value / @ConfigurationProperties

`@ImportResource` 主要用于：
- 遗留系统迁移过渡期
- 无法修改的第三方 XML 配置

---

## Q5：可以用 @ImportResource 导入 properties 文件吗？

**A**：不推荐，应该用 `@PropertySource`：

```java
// 不推荐
@ImportResource("classpath:config.properties")

// 推荐
@PropertySource("classpath:config.properties")
```

---

## Q6：导入的 XML 中的 Bean 会被 @Autowired 自动注入吗？

**A**：会的。XML 导入的 Bean 和普通 @Bean 定义一样，可以被 @Autowired 注入。
