# SpringApplication 构造器

## Q1：SpringApplication 构造器的主要作用是什么？

**A**：

```java
public SpringApplication(Class<?>... primarySources) {
    // 1. 保存主配置类
    this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    
    // 2. 判断 Web 应用类型
    this.webApplicationType = WebApplicationType.deduceFromClasspath();
    
    // 3. 加载初始化器
    this.setInitializers(this.getSpringFactoriesInstances(
        ApplicationContextInitializer.class));
    
    // 4. 加载监听器
    this.setListeners(this.getSpringFactoriesInstances(
        ApplicationListener.class));
    
    // 5. 确定 main 方法所在类
    this.mainApplicationClass = this.deduceMainApplicationClass();
}
```

**核心功能**：初始化 SpringApplication 的基本组件。

---

## Q2：primarySources 和 sources 有什么区别？

**A**：

| 属性 | 说明 | 来源 |
|------|------|------|
| `primarySources` | 主配置类（启动类） | 构造器参数 |
| `sources` | 所有配置源 | primarySources + 额外配置 |

**使用场景**：
```java
// primarySources：用于 Banner 显示、推断 main 类
// sources：用于加载 Bean 定义

SpringApplication app = new SpringApplication(App.class);
app.setSources(Collections.singleton("com.example.config.MyConfig"));
```

---

## Q3：WebApplicationType 是如何判断的？

**A**：

```java
// 源码简化
WebApplicationType.deduceFromClasspath() {
    if (存在 "org.springframework.web.reactive.DispatcherHandler" 
        && 不存在 "org.springframework.web.servlet.DispatcherServlet") {
        return REACTIVE;
    }
    if (存在 "javax.servlet.Servlet" 
        || 存在 "org.springframework.web.servlet.support.GenericWebApplicationContext") {
        return SERVLET;
    }
    return NONE;
}
```

**判断依据**：
```
REACTIVE ← Spring WebFlux 类存在
SERVLET  ← Servlet API 存在
NONE     ← 以上都不存在
```

---

## Q4：如何加载 spring.factories 中的组件？

**A**：通过 `getSpringFactoriesInstances()` 方法：

```java
private <T> List<T> getSpringFactoriesInstances(Class<T> type) {
    return SpringFactoriesLoader.forDefaultLanguageLocation()
                                .load(type)
                                .stream()
                                .map(factory -> instantiateFactory(factory))
                                .collect(Collectors.toList());
}

// spring.factories 文件示例
org.springframework.context.ApplicationContextInitializer=\
  org.springframework.boot.env.EnvironmentPostProcessor=\
  com.example.my.config.MyInitializer

org.springframework.context.ApplicationListener=\
  com.example.my.listener.MyListener
```

---

## Q5：mainApplicationClass 是如何推断的？

**A**：

```java
private Class<?> deduceMainApplicationClass() {
    try {
        StackTraceElement[] stackTrace = new RuntimeException().getStackTrace();
        for (StackTraceElement element : stackTrace) {
            if ("main".equals(element.getMethodName())) {
                // 返回包含 main 方法的类
                return Class.forName(element.getClassName());
            }
        }
    } catch (ClassNotFoundException ex) {
        // 忽略
    }
    return null;
}
```

**用途**：用于 Banner 显示。

---

## Q6：可以传递多个主配置类吗？

**A**：**可以**，但通常只传一个：

```java
// 多配置类
new SpringApplication(App.class, Config1.class, Config2.class);

// 内部合并到 primarySources
this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
```

**实际场景**：多配置类用于模块化设计：
```java
new SpringApplication(
    Application.class,
    DatabaseConfig.class,    // 数据库配置
    SecurityConfig.class,    // 安全配置
    RedisConfig.class        // Redis 配置
);
```

---

## Q7：ResourceLoader 有什么用？

**A**：`ResourceLoader` 用于加载类路径资源和文件资源。

```java
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    this.resourceLoader = resourceLoader;
    // ...
}

// 使用场景
ResourceLoader loader = getClass().getClassLoader();
Resource resource = loader.getResource("classpath:banner.txt");
```

**大多数情况下**：传入 `null`，使用默认的 `DefaultResourceLoader`。
