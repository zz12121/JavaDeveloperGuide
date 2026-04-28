# properties vs YAML 对比 QA

## Q1：Spring Boot 会同时加载 properties 和 YAML 吗？

**A**：会的。Spring Boot 按优先级依次加载：
1. `application.properties`
2. `application.yml`
3. `application-{profile}.properties`
4. `application-{profile}.yml`
5. 命令行参数

同一配置项，后面覆盖前面。

---

## Q2：项目里应该用 properties 还是 YAML？

**A**：没有绝对答案，建议：
- **新项目**：用 YAML，结构更清晰
- **旧项目**：保持一致，避免混用
- **团队协作**：统一规范最重要

---

## Q3：properties 的 `a.b.c` 和 YAML 的缩进，Java 读取有区别吗？

**A**：没有区别。Spring Boot 配置绑定时会统一处理，最终都映射到 `@ConfigurationProperties` 或 `@Value`。

```java
// 都能读取
@Value("${server.servlet.context-path}")
private String contextPath;
```

---

## Q4：可以同时使用 properties 和 YAML 吗？

**A**：可以，但不推荐。混用会导致维护困难：
- 开发环境用 YAML
- 生产环境用 properties
- 容易混淆配置来源

---

## Q5：IDEA 对 YAML 支持好吗？

**A**：专业版支持良好，包括：
- 语法高亮
- 自动补全（配合 spring-boot-configuration-processor）
- 格式检查
