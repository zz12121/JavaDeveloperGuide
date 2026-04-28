# @EnableAutoConfiguration vs @SpringBootApplication

## 先说结论

- `@EnableAutoConfiguration`：启用自动配置的**功能开关**
- `@SpringBootApplication`：主启动类的**标配组合注解**，包含自动配置

两者不是替代关系，`@SpringBootApplication` 已经内置了 `@EnableAutoConfiguration`。

## 深度解析

### 注解层级关系

```
┌─────────────────────────────────────────────────────────────┐
│                    注解层级关系                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   @SpringBootApplication                                    │
│   ├── @SpringBootConfiguration                              │
│   │   └── @Configuration                                   │
│   ├── @EnableAutoConfiguration ←── 这里                   │
│   │   ├── @AutoConfigurationPackage                        │
│   │   └── @Import(AutoConfigurationImportSelector.class)   │
│   └── @ComponentScan                                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### @EnableAutoConfiguration 源码

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
    String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";
    
    // 排除的自动配置类
    Class<?>[] exclude() default {};
    
    // 排除的自动配置类名
    String[] excludeName() default {};
}
```

### 使用场景对比

| 场景 | 使用 | 说明 |
|------|------|------|
| 主启动类 | `@SpringBootApplication` | 推荐，最完整 |
| 自定义配置类启用自动配置 | `@EnableAutoConfiguration` | 独立模块使用 |
| 测试类 | `@SpringBootTest` | 包含自动配置 |

### 自定义配置类中使用

```java
// 场景：某个独立模块需要启用 Spring Boot 自动配置
@Configuration
@EnableAutoConfiguration(exclude = {
    DataSourceAutoConfiguration.class  // 该模块不需要数据源
})
public class MyModuleConfig {
    // ...
}
```

## 易错点/踩坑

- ❌ 误以为 @EnableAutoConfiguration 可以替代 @SpringBootApplication
- ❌ 在非 Spring Boot 项目中使用 @EnableAutoConfiguration
- ❌ 排除自动配置时参数名写错

## 代码示例

### 主启动类（标准写法）

```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

### 手动启用自动配置

```java
@Configuration
@EnableAutoConfiguration
public class CustomConfig {
    // 这个配置类会启用自动配置
    // 但通常配合 @ComponentScan 使用才有意义
}
```

### @SpringBootTest 中的自动配置

```java
@SpringBootTest
// 等价于
// @ContextConfiguration(classes = SpringBootApplication.class)
// 自动包含 @EnableAutoConfiguration
class MyTest {
    @Test
    void test() {
        // 测试类中已经启用自动配置
    }
}
```

## 关联知识点

- `@SpringBootApplication` 组合注解
- 自动配置原理
- `@Import` 注解
- `AutoConfigurationImportSelector`
