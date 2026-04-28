# 自动配置条件注解概述

## Q1：Spring Boot 的条件注解和 Spring 的 @Conditional 有什么关系？

**A**：`@Conditional` 是 Spring 4.0 引入的元注解，Spring Boot 的所有 `@ConditionalOn*` 注解都基于它实现：

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Conditional {
    Class<? extends Condition>[] value();
}

// Spring Boot 扩展
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional(OnClassCondition.class)
public @interface ConditionalOnClass {
    Class<?>[] value() default {};
}
```

---

## Q2：条件注解可以叠加使用吗？

**A**：可以，叠加表示"且"的关系：

```java
@AutoConfiguration
@ConditionalOnClass(Xxx.class)        // AND
@ConditionalOnProperty(name = "my.enabled", havingValue = "true")  // AND
@ConditionalOnBean(Yyy.class)        // AND
public class MyAutoConfiguration { }
// 只有三个条件都满足，才会加载
```

---

## Q3：条件注解放在方法上 vs 类上有什么区别？

**A**：

| 位置 | 作用范围 | 效果 |
|------|----------|------|
| **类上** | 整个配置类 | 条件不满足，类中所有 @Bean 都不注册 |
| **方法上** | 单个 @Bean 方法 | 只有该方法不注册，其他方法正常 |

```java
@AutoConfiguration
@ConditionalOnClass(Xxx.class)  // ❌ 类级别失败，整个类不加载
public class MyAutoConfiguration {
    @Bean
    public A a() { return new A(); }

    @Bean
    public B b() { return new B(); }  // 也不会加载
}
```

```java
@AutoConfiguration
public class MyAutoConfiguration {
    @Bean
    @ConditionalOnClass(Xxx.class)  // ✅ 方法级别，只影响这一个 Bean
    public A a() { return new A(); }

    @Bean
    public B b() { return new B(); }  // 正常加载
}
```

---

## Q4：如何排查条件注解为什么不生效？

**A**：三种方法：

**1. 开启 debug 模式**
```properties
debug: true
```

**2. 查看自动配置报告**
```bash
curl http://localhost:8080/actuator/conditions
```

**3. 查看日志中的条件判断详情**
```
# 过滤 org.springframework.boot.autoconfigure.condition 包
logging.level.org.springframework.boot.autoconfigure.condition=DEBUG
```

---

## Q5：Spring Boot 有多少个 @ConditionalOn* 注解？

**A**：核心 10 个：

| 条件 | 作用 |
|------|------|
| `@ConditionalOnClass` | classpath 有指定类 |
| `@ConditionalOnMissingClass` | classpath 无指定类 |
| `@ConditionalOnBean` | 容器有指定 Bean |
| `@ConditionalOnMissingBean` | 容器无指定 Bean |
| `@ConditionalOnProperty` | 属性满足条件 |
| `@ConditionalOnResource` | 资源文件存在 |
| `@ConditionalOnWebApplication` | Web 环境 |
| `@ConditionalOnNotWebApplication` | 非 Web 环境 |
| `@ConditionalOnExpression` | SpEL 表达式为 true |
| `@ConditionalOnCloudPlatform` | 指定云平台 |

