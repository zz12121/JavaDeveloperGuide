# Spring Boot 2.7 自动配置变化

## 先说结论

Spring Boot 2.7 对自动配置机制做了重大升级：用 `AutoConfiguration.imports` 文件替代 `spring.factories`，并引入了 `AutoConfiguration` 注解替代 `@Configuration`，同时清理了大量废弃 API，是一次里程碑式的改进。

## 深度解析

### 主要变化一览

| 变化项 | 2.7 之前 | 2.7+ |
|--------|----------|------|
| 配置文件 | `spring.factories` | `AutoConfiguration.imports` |
| 注解 | `@Configuration` + `@Conditional*` | `@AutoConfiguration` |
| 排除配置 | `spring.autoconfigure.exclude` | 不变 |
| 推荐打包 | `spring-boot-maven-plugin` | 不变 |
| 废弃警告 | 无 | spring.factories 警告 |

### 新增 @AutoConfiguration 注解

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Configuration
@Indexed                                    // 新增：支持快速索引
public @interface AutoConfiguration {
    @AliasFor(annotation = Configuration.class)
    boolean proxyBeanMethods() default true;

    @AliasFor(annotation = Configuration.class)
    boolean enforceUniqueMethods() default true;
}
```

**优点**：
- 明确标识"这是自动配置类"而非普通配置类
- 支持 `@Indexed` 注解，编译时生成索引文件，加载更快
- `proxyBeanMethods` 默认 true（保持向后兼容）

### @AutoConfigurationPackage 新注解

```java
// 替代原来隐式扫描主启动类所在包的行为
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(AutoConfigurationPackages.Registrar.class)
public @interface AutoConfigurationPackage {
    String[] basePackages() default {};
    Class<?>[] basePackageClasses() default {};
}
```

### 配置位置变化

```
# 旧（2.7 之前）
META-INF/spring.factories
  └── EnableAutoConfiguration=...

# 新（2.7+）
META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
  └── 每行一个类名
```

### 兼容性策略

```java
// AutoConfigurationImportSelector 读取顺序
if (exists("AutoConfiguration.imports")) {
    // 优先使用新格式
    loadAutoConfigurationFromImportsFile();
} else {
    // 兼容旧格式（带废弃警告）
    loadAutoConfigurationFromSpringFactories();
}
```

## 易错点/踩坑

- ❌ **spring.factories 仍有废弃警告** — 尽快迁移到 imports
- ❌ **新项目仍用旧格式** — 2.7+ 新项目应直接使用新格式
- ❌ **@AutoConfiguration 和 @Configuration 混用** — 自动配置类应统一使用 @AutoConfiguration

## 关联知识点

- [[AutoConfiguration_imports文件]] — 新配置文件详解
- [[AutoConfigurationPackage注解]] — 新增的包扫描注解
- [[spring.factories文件]] — 旧格式回顾
