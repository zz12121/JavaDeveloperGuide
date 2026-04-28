# @AutoConfigurationPackage 注解

## Q1：@AutoConfigurationPackage 有什么用？

**A**：它将指定的包注册为"自动配置包"，使该包下的组件可以被扫描到：

```java
@AutoConfigurationPackage
public @interface AutoConfigurationPackage { }

// 效果：将标注类的包名注册到 AutoConfigurationPackages
// 例如：标注在 com.example.config.MyAuto
// → 注册包：com.example.config
```

---

## Q2：@AutoConfigurationPackage 和 @ComponentScan 有什么区别？

**A**：

| 对比项 | @AutoConfigurationPackage | @ComponentScan |
|--------|--------------------------|----------------|
| 作用 | 注册包到自动配置包列表 | 扫描并注册 Bean |
| 触发时机 | 导入时立即注册 | refresh() 阶段执行 |
| 配合使用 | 与 @Import 一起 | 可扫描 @AutoConfigurationPackage 注册的包 |

---

## Q3：如何同时注册多个包？

**A**：

```java
@AutoConfigurationPackage(basePackages = {
    "com.example.config",
    "com.example.properties"
})
public @interface AutoConfigurationPackage { }
```

或使用类引用：
```java
@AutoConfigurationPackage(basePackageClasses = {
    ConfigA.class,
    ConfigB.class
})
public @interface AutoConfigurationPackage { }
```

---

## Q4：Spring Boot 如何使用 @AutoConfigurationPackage？

**A**：

```java
// 主启动类
@SpringBootApplication
public class MyApplication { }

// 等效于：
@Configuration
@EnableAutoConfiguration
@ComponentScan  // Spring Boot 自动添加
public @interface SpringBootApplication { }

// 内部：@EnableAutoConfiguration 导入了 AutoConfigurationImportSelector
// 它会扫描所有标注了 @AutoConfigurationPackage 的包
```

---

## Q5：自定义 Starter 需要 @AutoConfigurationPackage 吗？

**A**：**不需要**。@AutoConfigurationPackage 主要用于主启动类的包扫描：

```java
// ❌ 自定义 Starter 通常不需要
@AutoConfigurationPackage
public class MyAutoConfiguration { }

// ✅ Starter 只需要声明 AutoConfiguration.imports
// META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
// com.example.mystarter.MyAutoConfiguration
```

