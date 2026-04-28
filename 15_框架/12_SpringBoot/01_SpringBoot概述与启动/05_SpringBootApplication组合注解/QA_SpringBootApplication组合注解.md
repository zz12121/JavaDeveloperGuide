# @SpringBootApplication 组合注解

## Q1：@SpringBootApplication 是哪三个注解的组合？

**A**：`@SpringBootApplication` 是 Spring Boot 1.x 推出的组合注解，等价于同时标注：

```java
@SpringBootApplication
// 等价于
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan
```

| 注解 | 来源 | 作用 |
|------|------|------|
| `@SpringBootConfiguration` | Spring Boot | 继承自 `@Configuration`，标记配置类 |
| `@EnableAutoConfiguration` | Spring Boot | 启用自动配置 |
| `@ComponentScan` | Spring | 组件扫描，默认扫描主启动类所在包 |

---

## Q2：@EnableAutoConfiguration 和 @SpringBootApplication 的区别？

**A**：

| 对比 | @EnableAutoConfiguration | @SpringBootApplication |
|------|-------------------------|----------------------|
| **类型** | 启用注解 | 组合注解 |
| **包含** | 单个功能 | 三个注解组合 |
| **使用场景** | 自定义配置类中启用自动配置 | 主启动类标配 |
| **等价值** | 必须配合 @Configuration 和 @ComponentScan 使用 | 独立使用即可 |

**等价的完整写法**：
```java
// 使用 @SpringBootApplication（推荐）
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}

// 等价于（不推荐，写法冗余）
@Configuration
@EnableAutoConfiguration
@ComponentScan
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

---

## Q3：@ComponentScan 的默认扫描规则是什么？

**A**：`@ComponentScan` 默认扫描**主启动类所在的包及其所有子包**。

```java
@ComponentScan  // 不指定参数
// 等价于
@ComponentScan(basePackages = "主启动类所在的包")
```

**扫描示例**：
```
项目结构：
com.example.demo/
├── DemoApplication.java    ← 主启动类
├── config/
│   └── AppConfig.java      ← @Configuration，被扫描 ✓
├── controller/
│   └── UserController.java ← @Controller，被扫描 ✓
├── service/
│   ├── UserService.java    ← @Service，被扫描 ✓
│   └── impl/
│       └── UserServiceImpl.java  ← 被扫描 ✓
└── mapper/
    └── UserMapper.java     ← @Mapper（MyBatis），被扫描 ✓
```

---

## Q4：如何排除某些自动配置类？

**A**：有三种方式：

**方式一：exclude 参数（常用）**
```java
@SpringBootApplication(exclude = {
    DataSourceAutoConfiguration.class,
    HibernateJpaAutoConfiguration.class
})
public class Application { }
```

**方式二：excludeName 参数**
```java
@SpringBootApplication(excludeName = {
    "org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration"
})
public class Application { }
```

**方式三：application.yml 配置**
```yaml
spring:
  autoconfigure:
    exclude:
      - org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
      - org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaAutoConfiguration
```

---

## Q5：@ComponentScan 和 @SpringBootApplication 中的 @ComponentScan 会冲突吗？

**A**：**不会冲突**，`@SpringBootApplication` 已经包含了 `@ComponentScan`。

```java
@SpringBootApplication
// 内部包含
@ComponentScan(excludeFilters = {
    @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
    @Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class)
})
```

**如果手动添加 @ComponentScan**：会覆盖默认行为！

```java
@SpringBootApplication
@ComponentScan("com.example.common")  // ❌ 覆盖了默认的包扫描
public class Application { }

// 正确做法：使用 scanBasePackages 参数
@SpringBootApplication(scanBasePackages = {
    "com.example.demo",
    "com.example.common"
})
public class Application { }
```

---

## Q6：@SpringBootApplication 中的 excludeFilters 是什么？

**A**：这是 `@ComponentScan` 的排除过滤器，排除两种类：

| 过滤器 | 作用 |
|--------|------|
| `TypeExcludeFilter` | 排除 Spring Boot 内部的类型排除类 |
| `AutoConfigurationExcludeFilter` | 排除重复的自动配置类 |

```java
@ComponentScan(excludeFilters = {
    @Filter(type = FilterType.CUSTOM, 
            classes = TypeExcludeFilter.class),      // Spring 内部排除
    @Filter(type = FilterType.CUSTOM, 
            classes = AutoConfigurationExcludeFilter.class)  // 排除重复自动配置
})
```

---

## Q7：Spring Boot 2.7+ 推荐的写法？

**A**：Spring Boot 2.7 开始，推荐**不要**在 `@SpringBootApplication` 上使用 `scanBasePackages`。

```java
// ✅ 推荐：保持简单
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}

// ✅ 需要多包扫描时：
@SpringBootApplication
public class Application { }

// 在单独的配置类中指定
@Configuration
@ComponentScan(basePackages = {"com.example.demo", "com.example.common"})
public class ScanConfig { }
```

**原因**：将主启动类与扫描配置分离，职责更清晰。
