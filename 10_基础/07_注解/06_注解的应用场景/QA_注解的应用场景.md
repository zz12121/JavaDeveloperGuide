---
title: 注解的应用场景
tags:
  - Java/注解
  - 场景型
  - 问答
module: 07_注解
created: 2026-04-18
---

# 注解的应用场景

## Q：注解在实际开发中有哪些应用场景？

**A：** 注解的核心作用是**声明式编程**，让框架通过反射读取注解并执行逻辑。常见场景：

1. **Spring 框架**：`@Component`、`@Autowired`、`@Transactional`、`@RequestMapping` 等，实现 IoC、DI、AOP
2. **参数校验**：`@NotNull`、`@Size`、`@Email` 等（JSR 303 Bean Validation）
3. **ORM 映射**：JPA 的 `@Entity`、`@Table`、`@Column`；MyBatis 的 `@Select`、`@Insert`
4. **单元测试**：JUnit 的 `@Test`、`@BeforeEach`、`@Disabled`
5. **代码简化**：Lombok 的 `@Data`、`@Builder`、`@Slf4j`（编译期注解处理器）

## Q：Spring 的 @Autowired 是怎么工作的？

**A：** `@Autowired` 是 `RUNTIME` 保留策略的注解。Spring 在启动阶段（Bean 实例化后、初始化前），通过反射扫描 Bean 的字段、构造方法、Setter 方法上的 `@Autowired` 注解，然后从 IoC 容器中查找匹配的 Bean 完成依赖注入。核心流程：**反射扫描注解 → 根据类型/名称匹配 → 注入依赖**。

## Q：Lombok 的注解和其他注解有什么不同？

**A：** Lombok 的注解（如 `@Data`）是在**编译期**由注解处理器（Annotation Processor）处理的，它直接修改抽象语法树（AST），在生成的 class 文件中已经包含了 getter/setter 等方法。而 Spring 的注解是在**运行时**通过反射读取处理的。两者原理完全不同。
