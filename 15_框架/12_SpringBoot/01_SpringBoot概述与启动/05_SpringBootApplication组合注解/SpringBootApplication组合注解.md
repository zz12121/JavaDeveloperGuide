# @SpringBootApplication 组合注解

## 先说结论

`@SpringBootApplication` 是 Spring Boot 的**核心组合注解**，等价于同时标注 `@Configuration`、`@EnableAutoConfiguration`、`@ComponentScan`，是主启动类的标配。

## 深度解析

### 注解源码

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = {
    @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
    @Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class)
})
public @interface SpringBootApplication {
    // ...
}
```

### 三层组合

```
@SpringBootApplication
│
├── @SpringBootConfiguration
│   └── @Configuration        → 标注当前类是配置类
│
├── @EnableAutoConfiguration
│   └── 启用 Spring Boot 自动配置
│
└── @ComponentScan
    └── 扫描 @Component/@Service/@Repository/@Controller
```

### 各注解详解

| 注解 | 作用 | 常用参数 |
|------|------|----------|
| `@Configuration` | 标记为配置类，替代 XML | - |
| `@EnableAutoConfiguration` | 启用自动配置 | exclude, excludeName |
| `@ComponentScan` | 组件扫描 | basePackages, includeFilters |

### @EnableAutoConfiguration 详解

```java
@EnableAutoConfiguration
// 等价于
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
```

**自动配置扫描**：
```
@EnableAutoConfiguration
    ↓
加载 spring-boot-autoconfigure.jar 中的自动配置类
    ↓
根据 @Conditional* 条件判断是否生效
    ↓
生效则注册对应的 Bean
```

### @ComponentScan 默认行为

```java
@ComponentScan
// 默认值
basePackages = "${componentScan.package}"
// 即主启动类所在的包

// 扫描范围
com.example.demo/
├── DemoApplication.java        ← 扫描起点
├── controller/                 ← ✓ 扫描
│   └── UserController.java     ← ✓ 扫描
├── service/                    ← ✓ 扫描
│   └── UserService.java        ← ✓ 扫描
└── entity/                     ← ✓ 扫描
    └── User.java               ← ✓ 扫描
```

## 易错点/踩坑

- ❌ `@SpringBootApplication` 和 `@ComponentScan` 混用导致扫描路径混乱
- ❌ 排除自动配置时使用错误参数（exclude vs excludeName）
- ❌ 主启动类不在根包，导致组件扫描不全

## 代码示例

### 标准用法

```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

### 排除指定自动配置

```java
@SpringBootApplication(exclude = {
    DataSourceAutoConfiguration.class,
    HibernateJpaAutoConfiguration.class
})
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

### 自定义扫描包

```java
@SpringBootApplication(scanBasePackages = {
    "com.example.demo",
    "com.example.common"
})
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

### 分开使用三个注解（效果相同）

```java
// 以下两种写法等价：
@SpringBootApplication
public class Application { }

// ↓ 等价于 ↓

@Configuration
@EnableAutoConfiguration
@ComponentScan
public class Application { }
```

## 图解/流程

```
┌─────────────────────────────────────────────────────────────┐
│              @SpringBootApplication 组合注解                  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   @SpringBootApplication                                    │
│   ┌───────────────────────────────────────────────────────┐ │
│   │                                                       │ │
│   │   @SpringBootConfiguration                           │ │
│   │   └── @Configuration                                  │ │
│   │       → 标记配置类，注册 @Bean 方法                    │ │
│   │                                                       │ │
│   │   @EnableAutoConfiguration                            │ │
│   │   └── @Import(AutoConfigurationImportSelector.class) │ │
│   │       → 加载 spring.factories 中的自动配置类          │ │
│   │                                                       │ │
│   │   @ComponentScan                                      │ │
│   │       → 默认扫描主启动类所在包及其子包                 │ │
│   │                                                       │ │
│   └───────────────────────────────────────────────────────┘ │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 关联知识点

- `@EnableAutoConfiguration` 详解
- 自动配置原理
- `@ComponentScan` 组件扫描
- `@Configuration` 配置类
