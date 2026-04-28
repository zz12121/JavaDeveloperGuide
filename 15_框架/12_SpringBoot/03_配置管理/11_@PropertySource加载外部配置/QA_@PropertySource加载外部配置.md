# @PropertySource 加载外部配置 QA

## Q1：@PropertySource 和 application.properties 同时存在，谁优先级高？

**A**：**后加载的优先级高**。

执行顺序：
1. `@PropertySource` 加载的文件
2. `application.properties`

所以 `application.properties` 中的同名配置会覆盖 `@PropertySource` 加载的。

---

## Q2：如何加载多个配置文件？

**A**：用 `@PropertySources` 或多个 `@PropertySource`：

```java
@PropertySources({
    @PropertySource("classpath:db.properties"),
    @PropertySource("classpath:redis.properties"),
    @PropertySource("classpath:mail.properties")
})
```

---

## Q3：配置文件不在 classpath 下怎么办？

**A**：用 `file:` 前缀：

```java
@PropertySource("file:./config/external.properties")
@PropertySource("file:/etc/myapp/config.properties")
```

---

## Q4：可以动态加载配置文件吗？

**A**：默认不支持动态加载。可以通过：
- `@RefreshScope` 部分支持
- 自定义 `PropertySourceLocator` 实现
- Spring Cloud Config

---

## Q5：@PropertySource 加载的配置文件支持占位符吗？

**A**：支持，但占位符只能引用已存在的属性：

```properties
# db.properties
db.driver=com.mysql.cj.jdbc.Driver
db.url=jdbc:mysql://localhost:3306/${db.name:myapp}
```

---

## Q6：Spring Boot 中 @PropertySource 不生效？

**A**：检查：
1. 是否在 @Configuration 类上使用
2. 路径是否正确
3. 是否有环境变量冲突
4. 考虑改用 `spring.config.import`
