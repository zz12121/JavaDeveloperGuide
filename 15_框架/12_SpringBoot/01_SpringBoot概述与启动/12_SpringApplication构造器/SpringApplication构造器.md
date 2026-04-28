# SpringApplication 构造器

## 先说结论

`SpringApplication` 构造器接收主配置类（primarySources）和可变参数 sources，用于确定应用类型、加载初始化器和监听器。

## 深度解析

### 构造器源码

```java
public class SpringApplication {
    
    // 主配置类
    private Set<Class<?>> primarySources;
    
    // 所有配置源
    private Set<String> sources = new LinkedHashSet<>();
    
    public SpringApplication(Class<?>... primarySources) {
        this(null, primarySources);
    }
    
    public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
        this.resourceLoader = resourceLoader;
        
        // 保存主配置类
        this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
        
        // 判断 Web 应用类型
        this.webApplicationType = WebApplicationType.deduceFromClasspath();
        
        // 加载 spring.factories 中的初始化器
        this.setInitializers(this.getSpringFactoriesInstances(
            ApplicationContextInitializer.class));
        
        // 加载 spring.factories 中的监听器
        this.setListeners(this.getSpringFactoriesInstances(
            ApplicationListener.class));
        
        // 确定主配置类所在的 main 方法
        this.mainApplicationClass = this.deduceMainApplicationClass();
    }
}
```

### 参数说明

| 参数 | 说明 | 典型值 |
|------|------|--------|
| `primarySources` | 主配置类数组 | `Application.class` |
| `resourceLoader` | 资源加载器 | 通常为 null |

### sources vs primarySources

| 属性 | 说明 | 使用场景 |
|------|------|----------|
| `primarySources` | 主启动类 | 用于 Banner 显示、推断 main 方法 |
| `sources` | 所有配置源 | 包括主类 + 额外配置 |

### 关键初始化

```java
// 1. 加载 spring.factories 中的初始化器
this.setInitializers(
    getSpringFactoriesInstances(ApplicationContextInitializer.class)
);

// 2. 加载 spring.factories 中的监听器
this.setListeners(
    getSpringFactoriesInstances(ApplicationListener.class)
);

// 3. 推断 main 方法所在类
this.mainApplicationClass = deduceMainApplicationClass();
```

## 易错点/踩坑

- ❌ 混淆 primarySources 和 sources 的用途
- ❌ 忘记传递主配置类导致扫描失败
- ❌ 自定义 sources 时覆盖了 primarySources

## 代码示例

### 标准用法

```java
// 内部调用
new SpringApplication(Application.class);

// 实际执行
public SpringApplication(Class<?>... primarySources) {
    this(null, primarySources);
}
```

### 传递多个配置类

```java
// 方式一：可变参数
new SpringApplication(Application.class, DatabaseConfig.class, SecurityConfig.class);

// 方式二：setSources
SpringApplication app = new SpringApplication(Application.class);
app.setSources(Collections.singleton("com.example.myapp.config"));
```

### 自定义资源加载器

```java
DefaultResourceLoader loader = new DefaultResourceLoader();
loader.setClassLoader(getClass().getClassLoader());

SpringApplication app = new SpringApplication(loader, Application.class);
```

## 关联知识点

- SpringApplication 静态方法
- `SpringApplication.run()` 启动流程
- spring.factories 加载
- ApplicationContextInitializer
