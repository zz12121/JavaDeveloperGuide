# spring.factories 文件

## 先说结论

`spring.factories` 是 Spring Boot 2.7 之前的自动配置元数据文件，位于 JAR 的 `META-INF/` 目录下，用于声明自动配置类列表。Spring Boot 2.7+ 已废弃此格式，改为 `AutoConfiguration.imports` 文件。

## 深度解析

### 文件位置

```
META-INF/spring.factories
```

### 文件格式

```properties
# 格式：键=值的逗号分隔列表
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\
  org.springframework.boot.autoconfigure.data.jpa.JpaRepositoriesAutoConfiguration,\
  org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration,\
  org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration
```

### 加载机制

```java
// SpringFactoriesLoader 核心加载逻辑
public static List<String> loadFactoryNames(
        Class<?> factoryType,     // 如 EnableAutoConfiguration.class
        ClassLoader classLoader) {

    Enumeration<URL> urls = classLoader.getResources(
        "META-INF/spring.factories");

    Properties properties = new Properties();
    while (urls.hasMoreElements()) {
        // 读取并合并多个 spring.factories
        properties.load(url.openStream());
    }

    String factoryClassName = factoryType.getName();
    String factoryNames = properties.getProperty(factoryClassName);
    // 返回逗号分隔的配置类列表
}
```

### 特点

| 特性 | 说明 |
|------|------|
| 位置固定 | 必须在 JAR 的 `META-INF/` 下 |
| 支持多文件 | 多个 JAR 可以各自声明，按 classpath 顺序合并 |
| Properties 格式 | `key=value` 对，value 支持多行 `\` 续行 |
| 无需编译 | 纯文本文件，随 JAR 发布 |

### spring.factories 其他用途

```properties
# 声明应用上下文初始化器
org.springframework.context.ApplicationContextInitializer=\
  org.springframework.boot.autoconfigure.SharedMetadataReaderFactoryContextInitializer

# 声明失败分析器
org.springframework.boot.SpringBootExceptionReporter=\
  org.springframework.boot.diagnostics.FailureAnalyzers

# 声明自动配置过滤器
org.springframework.boot.autoconfigure.AutoConfigurationImportFilter=\
  org.springframework.boot.autoconfigure.condition.OnClassCondition
```

## 易错点/踩坑

- ❌ **2.7+ 仍使用 spring.factories** — Spring Boot 会输出 Deprecation 警告，建议迁移
- ❌ **每行太长用 `\` 续行** — Windows 下换行符可能不兼容，建议拆分多行
- ❌ **多个 JAR 中声明同名 key** — 后加载的 JAR 会覆盖前者（不是追加），需注意 classpath 顺序
- ❌ **classLoader 为 null** — 在某些特殊 classLoader 场景下加载会失败

## 迁移到 AutoConfiguration.imports

```properties
# 旧格式 spring.factories
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  com.example.MyAutoConfiguration
```

```text
# 新格式 AutoConfiguration.imports（每行一个类）
com.example.MyAutoConfiguration
```

## 关联知识点

- [[AutoConfiguration_imports文件]] — 新版配置文件格式
- [[AutoConfigurationImportSelector]] — 读取配置文件的调用方
- [[SpringBoot2_7自动配置变化]] — 2.7 版本的重大变更
