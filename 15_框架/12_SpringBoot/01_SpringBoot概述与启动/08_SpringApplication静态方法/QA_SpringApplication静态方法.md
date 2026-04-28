# SpringApplication 静态方法

## Q1：SpringApplication.run() 和 new SpringApplication().run() 有什么区别？

**A**：

| 对比 | `SpringApplication.run()` | `new SpringApplication().run()` |
|------|-------------------------|-------------------------------|
| **写法** | 静态方法 | 实例方法 |
| **灵活性** | 固定 | 可自定义配置 |
| **返回类型** | ConfigurableApplicationContext | ConfigurableApplicationContext |

**源码对比**：
```java
// 静态 run 方法内部就是 new + run
public static ConfigurableApplicationContext run(Class<?> primarySource, String... args) {
    return new SpringApplication(primarySource).run(args);
}
```

---

## Q2：如何自定义 SpringApplication 的配置？

**A**：使用实例方法：

```java
public static void main(String[] args) {
    // 1. 创建实例
    SpringApplication app = new SpringApplication(DemoApplication.class);
    
    // 2. 自定义配置
    app.setBannerMode(Banner.Mode.OFF);           // 关闭 Banner
    app.setWebApplicationType(WebApplicationType.SERVLET);  // 设置 Web 类型
    app.setAddArguments(args -> {                 // 添加参数
        System.out.println("启动参数：" + Arrays.toString(args.getSourceArgs()));
    });
    
    // 3. 启动
    app.run(args);
}
```

---

## Q3：SpringApplication 有哪些常用的自定义配置？

**A**：

| 配置 | 方法 | 示例 |
|------|------|------|
| Banner | `setBannerMode()` | `app.setBannerMode(Banner.Mode.OFF)` |
| Web 类型 | `setWebApplicationType()` | `app.setWebApplicationType(WebApplicationType.NONE)` |
| 初始化器 | `addInitializers()` | `app.addInitializers(new MyInitializer())` |
| 监听器 | `addListeners()` | `app.addListeners(new MyListener())` |
| 配置源 | `setSources()` | `app.setSources(Collections.singleton("classpath:app.xml"))` |
| shutdown hook | `setRegisterShutdownHook()` | `app.setRegisterShutdownHook(false)` |

---

## Q4：setWebApplicationType() 如何使用？

**A**：设置应用类型：

```java
// 强制设置为非 Web 应用
app.setWebApplicationType(WebApplicationType.NONE);

// 设置为 Servlet Web 应用
app.setWebApplicationType(WebApplicationType.SERVLET);

// 设置为响应式 Web 应用
app.setWebApplicationType(WebApplicationType.REACTIVE);
```

**自动检测顺序**：
```
1. 检查 REACTIVE（Spring WebFlux）
2. 检查 SERVLET（Spring MVC）
3. 默认为 SERVLET
```

---

## Q5：如何获取启动后的 ApplicationContext？

**A**：

```java
public static void main(String[] args) {
    // 方式一：接收返回值
    ConfigurableApplicationContext ctx = 
        SpringApplication.run(Application.class, args);
    
    // 方式二：实例 + run
    SpringApplication app = new SpringApplication(Application.class);
    ConfigurableApplicationContext ctx = app.run(args);
    
    // 使用 context
    MyService service = ctx.getBean(MyService.class);
    service.doSomething();
}
```

---

## Q6：SpringApplication.create() 是什么？

**A**：`create()` 是 Spring Boot 2.4+ 提供的工厂方法，等价于 `new SpringApplication()`：

```java
// 2.4 之前
SpringApplication app = new SpringApplication(Application.class);

// 2.4+ 可以这样写
SpringApplication app = SpringApplication.create(Application.class);
```

**结合 Builder 使用**：
```java
SpringApplication app = SpringApplication
    .create(Application.class)
    .setBannerMode(Banner.Mode.OFF);

app.run(args);
```
