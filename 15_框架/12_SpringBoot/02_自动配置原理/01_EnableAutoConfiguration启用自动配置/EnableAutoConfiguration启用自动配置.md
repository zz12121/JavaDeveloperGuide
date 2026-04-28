# EnableAutoConfiguration 启用自动配置

## 先说结论

`@EnableAutoConfiguration` 是 Spring Boot 自动配置的入口注解，通过 `AutoConfigurationImportSelector` 从配置文件加载自动配置类列表，配合条件注解实现"按需加载"。它通常不单独使用，而是作为 `@SpringBootApplication` 组合注解的一部分出现。

## 深度解析

### 核心概念

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(AutoConfigurationImportSelector.class)   // 关键：导入选择器
public @interface EnableAutoConfiguration {

    // 排除指定自动配置类
    Class<?>[] exclude() default {};

    // 排除指定自动配置类（全限定名）
    String[] excludeName() default {};
}
```

### 工作原理

```
@EnableAutoConfiguration
        │
        ▼
@Import(AutoConfigurationImportSelector.class)
        │
        ▼
AutoConfigurationImportSelector
    └── DeferredImportSelector 接口实现
        │
        ▼
getAutoConfigurationEntry() → 获取候选配置列表
        │
        ▼
过滤 + 条件判断 → 最终生效的自动配置
```

### 与 @SpringBootApplication 的关系

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Configuration
@EnableAutoConfiguration          // ← 内置于此
@ComponentScan
public @interface SpringBootApplication {
    Class<?>[] exclude() default {};
    String[] excludeName() default {};
    String[] scanBasePackages() default {};
    Class<?>[] scanBasePackageClasses() default {};
}
```

### exclude / excludeName 用法

```java
// 方式一：exclude（排除具体类）
@SpringBootApplication(exclude = { DataSourceAutoConfiguration.class })

// 方式二：excludeName（排除全限定类名字符串）
@SpringBootApplication(excludeName = "org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration")

// 方式三：配置文件（最推荐，便于修改）
// application.yml
spring:
  autoconfigure:
    exclude:
      - org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```

## 易错点/踩坑

- ❌ **exclude 类名写错** — 排除不存在的类会抛 `IllegalStateException`
- ❌ **exclude 和 excludeName 混用不当** — `exclude` 优先级高于 `excludeName`
- ❌ **忘记加 `@SpringBootApplication` 就想单独用** — 需要配合 `@Configuration` 才能生效

## 代码示例

```java
// 单独使用 @EnableAutoConfiguration（不推荐，通常用 @SpringBootApplication）
@Configuration
@EnableAutoConfiguration(exclude = { WebMvcAutoConfiguration.class })
public class MyAppConfig {
    public static void main(String[] args) {
        SpringApplication.run(MyAppConfig.class, args);
    }
}
```

## 关联知识点

- [[SpringBootApplication组合注解]] — @SpringBootApplication 如何组合三个注解
- [[AutoConfigurationImportSelector]] — 导入选择器核心实现
- [[自动配置排除详解]] — 排除自动配置的多种方式
