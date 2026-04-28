# @ConditionalOnJndi

## Q1：@ConditionalOnJndi 的作用是什么？

**A**：只有当指定的 JNDI 资源在 JNDI 上下文中存在时，自动配置类才生效。

```java
@ConditionalOnJndi("java:comp/env/jdbc/myDS")
```

---

## Q2：什么时候会用到这个注解？

**A**：主要用于 Java EE 应用服务器环境（Tomcat、WildFly 等），检测 JNDI 资源：

- 外部容器提供的 DataSource
- JMS 连接工厂
- EJB 引用

Spring Boot 应用中很少使用，因为默认使用内嵌数据源。

---

## Q3：@ConditionalOnJndi 和 @ConditionalOnBean 的区别？

**A**：

| 条件 | @ConditionalOnJndi | @ConditionalOnBean |
|------|-------------------|-------------------|
| 检测目标 | JNDI 命名空间 | Spring IOC 容器 |
| 使用场景 | Java EE 容器 | Spring 应用 |
| 典型用途 | 外部容器资源 | Spring Bean |

---

## Q4：为什么 Spring Boot 自动配置很少用这个注解？

**A**：
1. **Spring Boot 推崇嵌入式**：默认使用内嵌 H2/Derby/MySQL
2. **JNDI 配置复杂**：需要容器配置 + 代码配置
3. **可移植性差**：依赖特定容器

只有特定场景（如 WAS 部署）才需要。
