# @AutoConfiguration 注解

## 先说结论

`@AutoConfiguration` 是 Spring Boot 2.4+ 引入的注解，用于替代 `@Configuration` 标注自动配置类。它继承 `@Configuration` 的所有特性，同时新增了对自动配置场景的优化支持。

## 深度解析

### 注解源码

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Configuration
@Indexed
public @interface AutoConfiguration {
    // 与 @Configuration.proxyBeanMethods 相同
    @AliasFor(annotation = Configuration.class)
    boolean proxyBeanMethods() default true;

    // 自动配置类专用
    @AliasFor(annotation = Configuration.class)
    boolean enforceUniqueMethods() default true;
}
```

### @Configuration 和 @AutoConfiguration 对比

| 特性 | @Configuration | @AutoConfiguration |
|------|---------------|-------------------|
| 核心功能 | 标记配置类 | 标记自动配置类 |
| @Indexed 支持 | 需要手动添加 | 自动支持 |
| proxyBeanMethods | 默认 true | 默认 true |
| 语义 | 业务配置 | 框架预置配置 |
| Spring Boot 版本 | 3.x | 2.4+（推荐） |

### 使用示例

```java
// Spring Boot 2.4+ 推荐写法
@AutoConfiguration
@ConditionalOnClass(DataSource.class)
@EnableConfigurationProperties(DataSourceProperties.class)
public class DataSourceAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public DataSource dataSource(DataSourceProperties properties) {
        return properties.initializeDataSourceBuilder().build();
    }
}
```

## 易错点/踩坑

- ❌ **Spring Boot 2.3 之前没有此注解** — 低版本只能使用 `@Configuration`
- ❌ **`proxyBeanMethods = false`** — 禁用后，同类内 @Bean 调用不会走代理，可能导致问题

## 关联知识点

- [[SpringBoot2_7自动配置变化]] — 2.7 版本变化
- [[自动配置条件注解概述]] — 条件注解体系
