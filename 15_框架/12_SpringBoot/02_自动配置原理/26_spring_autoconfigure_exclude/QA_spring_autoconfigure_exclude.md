# spring.autoconfigure.exclude 排除配置

## Q1：配置文件和注解方式哪个优先级高？

**A**：**配置文件的优先级更高**：

```java
// 注解方式排除
@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)
public class MyApp { }
```

```yaml
# 配置文件方式（覆盖注解）
spring:
  autoconfigure:
    exclude:
      - DataSourceAutoConfiguration  # 不会生效
```

```yaml
# 配置文件方式不排除（恢复注解的排除效果）
spring:
  autoconfigure:
    exclude: []  # 清空列表，恢复 DataSourceAutoConfiguration
```

---

## Q2：可以用通配符批量排除吗？

**A**：**不可以**，必须列出具体的类全限定名：

```properties
# ❌ 不支持通配符
spring.autoconfigure.exclude=org.springframework.boot.autoconfigure.data.*.DataAutoConfiguration

# ✅ 必须逐个列出
spring.autoconfigure.exclude=\
  org.springframework.boot.autoconfigure.data.jdbc.JdbcRepositoriesAutoConfiguration,\
  org.springframework.boot.autoconfigure.data.jpa.JpaRepositoriesAutoConfiguration
```

---

## Q3：排除的类不存在会报错吗？

**A**：**不会**，Spring Boot 会静默忽略不存在的类名：

```properties
# 这个不存在的类会被忽略，不会报错
spring.autoconfigure.exclude=com.nonexistent.AutoConfiguration
```

---

## Q4：如何在测试中临时排除某个配置？

**A**：

```java
@SpringBootTest(properties = {
    "spring.autoconfigure.exclude=org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration"
})
public class MyTest { }
```

---

## Q5：可以通过环境变量排除吗？

**A**：**可以**：

```bash
# Linux/Mac
export SPRING_AUTOCONFIGURE_EXCLUDE=org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
java -jar app.jar

# Windows PowerShell
$env:SPRING_AUTOCONFIGURE_EXCLUDE="org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration"
java -jar app.jar
```

