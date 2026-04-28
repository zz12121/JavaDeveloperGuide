# @ConditionalOnResource 条件判断

## 先说结论

`@ConditionalOnResource` 用于检查 classpath 或文件系统上是否存在指定的资源文件（如配置文件、properties 文件等），当资源存在时自动配置才生效。

## 深度解析

### 注解源码

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional(OnResourceCondition.class)
public @interface ConditionalOnResource {
    String[] resources() default {};
}
```

### 资源路径格式

```java
// classpath 资源
@ConditionalOnResource(resources = "classpath:applicationContext.xml")

// 文件系统资源
@ConditionalOnResource(resources = "file:/path/to/config.properties")

// Ant 风格路径（支持通配符）
@ConditionalOnResource(resources = "classpath*:META-INF/spring/*.xml")
```

### 实际应用场景

| 场景 | 资源类型 |
|------|----------|
| 存在 mybatis-config.xml 时启用 MyBatis 配置 | 文件系统 / classpath |
| 存在 ehcache.xml 时启用 EhCache | classpath |
| 存在 logback.xml 时启用日志配置 | classpath |

### 代码示例

```java
// 只有存在 applicationContext-datasource.xml 才加载数据源配置
@AutoConfiguration
@ConditionalOnResource(resources = "classpath:applicationContext-datasource.xml")
public class ExternalDataSourceAutoConfiguration {

    @Bean
    public DataSource externalDataSource() {
        // 加载外部配置文件
        return new ExternalDataSource();
    }
}
```

## 易错点/踩坑

- ❌ **classpath: 前缀** — 资源路径必须明确前缀，Spring 不会自动识别
- ❌ **resources 是 String[]** — 数组中的多个资源是 AND 关系，都存在才通过

## 关联知识点

- [[ConditionalOnClass条件判断]] — classpath 类检查
- [[自动配置条件注解概述]] — 条件注解体系
