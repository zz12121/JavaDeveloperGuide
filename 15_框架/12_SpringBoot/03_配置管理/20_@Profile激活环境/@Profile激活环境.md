# @Profile 激活环境

## 先说结论

`@Profile` 是 Spring 的环境隔离注解，可以标注在配置类或 Bean 上，只有激活指定 profile 时才生效。

## 深度解析

### 基本用法

```java
// 标注在类上
@Profile("dev")
@Configuration
public class DevDataSourceConfig {
    @Bean
    public DataSource dataSource() {
        return new EmbeddedDatabaseBuilder().build();
    }
}

// 标注在方法上
@Profile("prod")
@Bean
public DataSource prodDataSource() {
    return new HikariDataSource(prodConfig);
}
```

### 激活条件

```java
// 方式一：@Profile + @Profile
@Profile("dev")
@Profile("!prod")  // 排除 prod，等价于 "dev 且非 prod"

// 方式二：数组形式
@Profile({"dev", "test"})

// 方式三：表达式
@Profile("dev & !integration")
```

### 与 @ConfigurationProperties 配合

```java
@Profile("prod")
@ConfigurationProperties(prefix = "app.prod")
public class ProdProperties { }
```

### Spring Boot 中的常见用法

```java
// 数据源配置
@Profile("dev")
@Bean
public DataSource devDataSource() { ... }

@Profile("prod")
@Bean
public DataSource prodDataSource() { ... }

// 异步配置
@Profile("async")
@EnableAsync
public class AsyncConfig { }

// 安全配置
@Profile("secure")
@EnableWebSecurity
public class SecurityConfig { }
```

## 易错点/踩坑

- ❌ profile 未激活 — Bean 不会被创建
- ❌ 同名 Bean 冲突 — 不同 profile 的同名 Bean 不会冲突
- ❌ @Profile 和 spring.profiles.include 混淆 — @Profile 控制 Bean，include 加载额外 profile

## 关联知识点

- [[19_多环境配置]] — 环境配置理念
- [[21_spring_profiles_active激活]] — 激活方式
- [[22_application_profile配置文件]] — profile 文件
