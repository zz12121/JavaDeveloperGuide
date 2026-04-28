# 自动配置原理

## Q1：Spring Boot 是如何实现自动配置的？

**A**：通过 `@EnableAutoConfiguration` 注解触发，核心流程如下：

1. **主启动类**标注 `@SpringBootApplication` → 内含 `@EnableAutoConfiguration`
2. `AutoConfigurationImportSelector` 实现 `DeferredImportSelector`，在配置类处理阶段被调用
3. 从 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 读取自动配置类全限定名列表（2.7+），旧版本从 `META-INF/spring.factories` 读取
4. `AutoConfigurationImportFilter` 过滤不匹配的配置
5. 对每个配置类逐个执行 `@Conditional*` 条件判断
6. 通过条件 → 注册为 `BeanDefinition`；未通过 → 静默跳过
7. 最终加载几十到上百个自动配置类，完成 Spring Boot "零配置" 启动

---

## Q2：`@EnableAutoConfiguration` 和 `@SpringBootApplication` 是什么关系？

**A**：`@SpringBootApplication` 是一个组合注解，等价于同时标注以下三个注解：

| 组合注解 | 作用 |
|----------|------|
| `@Configuration` | 标注当前类是配置类（Spring 基础注解） |
| `@EnableAutoConfiguration` | **启用 Spring Boot 自动配置**（核心） |
| `@ComponentScan` | 扫描同包及子包下的组件（默认 basePackages="主启动类所在包"） |

```java
// @SpringBootApplication 的源码等价于：
@Configuration
@EnableAutoConfiguration
@ComponentScan
public @interface SpringBootApplication { }
```

---

## Q3：自动配置和手动配置哪个优先？

**A**：**用户手动配置 > 自动配置**

- `@ConditionalOnMissingBean` 的设计初衷就是"如果用户已经手动配置了，就不重复配置"
- Spring Boot 建议将自动配置类放在 `spring-boot-autoconfigure` 中，用户自定义配置放在业务代码中
- 可以通过 `spring.autoconfigure.exclude` 属性排除指定的自动配置类

---

## Q4：自动配置为什么有时候不生效？

**A**：常见原因如下：

| 原因 | 说明 | 排查方法 |
|------|------|----------|
| 依赖不存在 | `@ConditionalOnClass` 的类不在 classpath | 检查 pom.xml |
| Bean 已存在 | `@ConditionalOnMissingBean` 条件失败 | 检查是否有重复 Bean |
| 属性不匹配 | `@ConditionalOnProperty` 不满足 | 检查 application.yml |
| Web 环境不符 | 非 Web 环境不加载 Web 相关配置 | 检查应用类型 |
| 加载顺序问题 | 自动配置类加载时所需 Bean 尚未注册 | 调整 `@AutoConfigureOrder` |

---

## Q5：如何查看哪些自动配置生效了？

**A**：三种方式：

**方式一：启动日志（最简单）**
```bash
# application.yml 开启详细日志
debug: true
# 启动应用后，控制台输出自动配置报告
```

**方式二：Actuator 端点（生产可用）**
```bash
GET /actuator/conditions
GET /actuator/autoconfig
```

**方式三：手动查看报告 Bean**
```java
@Autowired
private ApplicationContext context;

@PostConstruct
public void showReport() {
    ConditionEvaluationReport report = ConditionEvaluationReport
        .get(context.getBeanFactory());
    System.out.println("positiveMatches: " + report.getPositiveMatches());
    System.out.println("negativeMatches: " + report.getNegativeMatches());
}
```

---

## Q6：如何排除不需要的自动配置？

**A**：三种方式，按优先级从高到低：

```java
// 方式一：主启动类排除
@SpringBootApplication(exclude = { DataSourceAutoConfiguration.class })
```

```java
// 方式二：excludeName（用全限定类名）
@SpringBootApplication(excludeName = "org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration")
```

```properties
# 方式三：配置文件排除（Spring Boot 2.x）
spring.autoconfigure.exclude=\
  org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration

# Spring Boot 1.x 语法不同
spring.autoconfigure.exclude=\
  org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```

---

## Q7：自定义 Starter 需要哪些自动配置？

**A**：一个完整的 Starter 需要以下结构：

```
my-spring-boot-starter/
├── src/main/resources/
│   └── META-INF/
│       └── spring/
│           └── org.springframework.boot.autoconfigure.AutoConfiguration.imports
│   # 或 2.7 之前用 spring.factories
│   └── META-INF/spring.factories
└── src/main/java/
    └── com.example/
        └── mystarter/
            ├── MyAutoConfiguration.java    # 自动配置类
            └── MyProperties.java            # 配置属性类（可选）
```

```java
// MyAutoConfiguration.java
@Configuration
@ConditionalOnClass(MyService.class)           // 依赖检查
@ConditionalOnProperty(name = "my.enabled", havingValue = "true", matchIfMissing = true)
@EnableConfigurationProperties(MyProperties.class)
public class MyAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public MyService myService(MyProperties props) {
        return new MyServiceImpl(props);
    }
}
```

---

## Q8：自动配置类的加载顺序如何控制？

**A**：三种方式：

| 方式 | 用法 | 说明 |
|------|------|------|
| `@AutoConfigureBefore` | 在指定配置类之前加载 | 隐式依赖 |
| `@AutoConfigureAfter` | 在指定配置类之后加载 | 显式依赖 |
| `@AutoConfigureOrder` | 指定具体顺序值 | 覆盖默认值（100） |

```java
@Configuration
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE + 10)  // 比默认更先加载
public class MyFirstAutoConfiguration {
    // ...
}
```

---

## Q9：Spring Boot 2.7 对自动配置做了哪些改变？

**A**：主要变化如下：

| 变化点 | 旧方式（2.7 之前） | 新方式（2.7+） |
|--------|--------------------|----------------|
| 配置位置 | `META-INF/spring.factories` | `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` |
| 注解格式 | `org.springframework.boot.autoconfigure.EnableAutoConfiguration` | 每行一个全限定类名，无键名 |
| 排除方式 | exclude 属性 | exclude + excludeName |
| `spring.factories` | 仍支持但显示警告 | **废弃**，建议迁移 |

**迁移建议**：将 `spring.factories` 中的自动配置类迁移到 `AutoConfiguration.imports` 文件。

---

## Q10：条件注解那么多，如何记忆？

**A**：分类记忆：

```
@ConditionalOnXxx     → Xxx 存在时才加载
@ConditionalOnMissingXxx → Xxx 不存在时才加载

-Xxx 可以是：
  - Class（类）
  - Bean（Bean 实例）
  - Property（配置属性）
  - Resource（资源文件）
  - WebApplication（Web 环境）
  - Expression（SpEL 表达式）
```

