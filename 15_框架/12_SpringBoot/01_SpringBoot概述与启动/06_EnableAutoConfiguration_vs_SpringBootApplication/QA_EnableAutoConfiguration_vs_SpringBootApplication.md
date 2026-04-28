# @EnableAutoConfiguration vs @SpringBootApplication

## Q1：@EnableAutoConfiguration 和 @SpringBootApplication 有什么区别？

**A**：

| 对比项 | @EnableAutoConfiguration | @SpringBootApplication |
|--------|-------------------------|------------------------|
| **类型** | 功能开关 | 组合注解 |
| **作用** | 启用自动配置 | = @Configuration + @EnableAutoConfiguration + @ComponentScan |
| **使用场景** | 自定义配置类 | 主启动类 |
| **能否单独使用** | 需要配合 @Configuration | 可以直接使用 |

**关系**：`@SpringBootApplication` 内部已经包含了 `@EnableAutoConfiguration`

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@SpringBootConfiguration
@EnableAutoConfiguration  // ← 内置了
@ComponentScan
public @interface SpringBootApplication { }
```

---

## Q2：什么时候需要单独使用 @EnableAutoConfiguration？

**A**：以下场景可能需要：

| 场景 | 示例 |
|------|------|
| **非主启动类的模块配置** | 自定义 Starter 中的配置类 |
| **条件性启用自动配置** | 根据环境变量决定是否启用 |
| **需要排除特定自动配置** | exclude 参数 |

**示例 - 自定义 Starter**：
```java
// 在 spring-boot-starter-xxx 的自动配置类中
@Configuration
@EnableAutoConfiguration(exclude = {
    DataSourceAutoConfiguration.class  // 该 Starter 不需要数据源
})
@ConditionalOnClass(XXXService.class)
public class XXXAutoConfiguration {
    @Bean
    @ConditionalOnMissingBean
    public XXXService xxxService() {
        return new XXXService();
    }
}
```

---

## Q3：@EnableAutoConfiguration 如何生效？

**A**：通过 `@Import` 导入 `AutoConfigurationImportSelector` 实现：

```java
@EnableAutoConfiguration
│
└── @Import(AutoConfigurationImportSelector.class)
        │
        └── 实现了 DeferredImportSelector 接口
                │
                └── Spring Boot 启动时调用 getAutoConfigurationEntry()
                        │
                        └── 加载 spring.factories 或 AutoConfiguration.imports
                                │
                                └── 返回需要导入的自动配置类列表
```

**源码流程**：
```java
// AutoConfigurationImportSelector.java
@Override
public String[] selectImports(AnnotationMetadata annotationMetadata) {
    if (!isEnabled(annotationMetadata)) {
        return EMPTY_IMPORTS;
    }
    
    // 获取自动配置类
    AutoConfigurationEntry autoConfigurationEntry = 
        getAutoConfigurationEntry(annotationMetadata);
    
    return autoConfigurationEntry.getConfigurations()
                                 .toArray(new String[0]);
}
```

---

## Q4：@EnableAutoConfiguration 的 exclude 参数如何使用？

**A**：有三种等价的排除方式：

```java
// 方式一：exclude 参数（Class）
@EnableAutoConfiguration(exclude = DataSourceAutoConfiguration.class)

// 方式二：excludeName 参数（类名）
@EnableAutoConfiguration(excludeName = 
    "org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration")

// 方式三：application.yml 配置
spring:
  autoconfigure:
    exclude:
      - org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```

**在 @SpringBootApplication 中**：
```java
// @SpringBootApplication 继承自 @EnableAutoConfiguration
@SpringBootApplication(exclude = {
    DataSourceAutoConfiguration.class,
    HibernateJpaAutoConfiguration.class
})
public class Application { }
```

---

## Q5：如何禁用 @EnableAutoConfiguration？

**A**：有以下几种方式：

**方式一：配置文件**
```yaml
spring:
  boot:
    enableautoconfiguration: false
```

**方式二：环境变量**
```bash
export SPRING_BOOT_ENABLEAUTOCONFIGURATION=false
```

**方式三：JVM 参数**
```bash
java -jar app.jar -Dspring.boot.enableautoconfiguration=false
```

---

## Q6：@EnableAutoConfiguration 和 @ComponentScan 可以同时使用吗？

**A**：**不建议同时使用**，`@SpringBootApplication` 已经包含了 `@ComponentScan`。

```java
// ❌ 冗余，不推荐
@Configuration
@EnableAutoConfiguration
@ComponentScan("com.example")  // 重复
public class Application { }

// ✅ 等价于上面，使用 @SpringBootApplication
@SpringBootApplication
public class Application { }
```

---

## Q7：@SpringBootApplication 是如何代替 @EnableAutoConfiguration 的？

**A**：`@SpringBootApplication` 的元注解中包含 `@EnableAutoConfiguration`：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@SpringBootConfiguration
@EnableAutoConfiguration    // ← 这里包含了
@ComponentScan
public @interface SpringBootApplication { }
```

**所以在主启动类上标注 `@SpringBootApplication` 就等于同时标注了**：
- `@Configuration`（通过 `@SpringBootConfiguration`）
- `@EnableAutoConfiguration`
- `@ComponentScan`
