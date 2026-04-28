# @SpringBootApplication注解三合一

## 先说结论

`@SpringBootApplication` 是一个组合注解，等价于 `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan`。**这是 SpringBoot 零配置的核心**。

## 深度解析

### 核心概念

```java
@SpringBootApplication 源码：
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Configuration        // 标注为配置类
@EnableAutoConfiguration  // 启用自动配置
@ComponentScan        // 组件扫描
public @interface SpringBootApplication {}
```

### 三个注解各自作用

| 注解 | 作用 |
|------|------|
| `@Configuration` | 标记为配置类，等价于 XML 中的 `<beans>` |
| `@EnableAutoConfiguration` | 启用 SpringBoot 自动配置机制 |
| `@ComponentScan` | 默认扫描主类所在包及子包 |

### @ComponentScan 默认行为

```
@SpringBootApplication 所在类
        ↓
包路径：com.example.demo
        ↓
自动扫描：com.example.demo.**
        ↓
所有标注 @Component/@Service/@Repository/@Controller/@Configuration 的类
        ↓
注册为 Spring Bean
```

### @EnableAutoConfiguration 细节

```java
@EnableAutoConfiguration 源码：
@AutoConfigurationPackage  // 自动配置类所在包
@Import(AutoConfigurationImportSelector.class)  // 导入自动配置选择器
public @interface EnableAutoConfiguration {
    String[] exclude() default {};
    String[] excludeName() default {};
}
```

## 易错点/踩坑

- ❌ 把 `@SpringBootApplication` 放在不正确的包位置 → 组件扫描漏掉某些 Bean
- ❌ 自定义包名后没调整主类位置 → 组件扫描不完整
- ❌ 同时使用 `@SpringBootApplication` + `@EnableAutoConfiguration` → 重复，浪费

## 代码示例

### 自定义扫描包

```java
// 方式1：excludeFilters 排除扫描
@SpringBootApplication
@ComponentScan(excludeFilters = @ComponentScan.Filter(type = FilterType.REGEX, pattern = "com.example.demo.external.*"))

// 方式2：使用 @ComponentScans 指定多个扫描路径
@ComponentScans({
    @ComponentScan(basePackages = "com.example.demo"),
    @ComponentScan(basePackages = "com.example.common")
})
public class Application {}
```

### 排除自动配置类

```java
@SpringBootApplication(exclude = {DataSourceAutoConfiguration.class})
public class Application {}
```

## 图解/流程

```
启动类标注 @SpringBootApplication
        ↓
┌─────────────────────────────────────┐
│ @Configuration                       │
│     → 标注配置类，@Bean 方法生效       │
├─────────────────────────────────────┤
│ @EnableAutoConfiguration             │
│     → 导入 AutoConfigurationImportSelector │
│     → 加载 spring-boot-autoconfigure │
│     → 按条件筛选，应用自动配置         │
├─────────────────────────────────────┤
│ @ComponentScan                       │
│     → 扫描当前包及子包的组件           │
│     → 注册为 Spring Bean              │
└─────────────────────────────────────┘
        ↓
应用启动完成
```

## 关联知识点

- `01_SpringBoot自动配置WebMvc`：WebMvc 自动配置来源
- `02_自动配置原理`：自动配置详细机制
