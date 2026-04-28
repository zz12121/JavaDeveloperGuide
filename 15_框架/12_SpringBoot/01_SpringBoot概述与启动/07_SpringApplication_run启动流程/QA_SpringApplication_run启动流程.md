# SpringApplication.run() 启动流程

## Q1：SpringApplication.run() 的核心流程是什么？

**A**：`SpringApplication.run()` 主要分为两个阶段：

**阶段一：创建 SpringApplication**
```java
new SpringApplication(Application.class)
    ├── 判断应用类型（REACTIVE/SERVLET/NONE）
    ├── 加载 spring.factories
    └── 初始化 initializers 和 listeners
```

**阶段二：执行 run()**
```java
run(args)
    ├── StopWatch.start()
    ├── BootstrapContext 创建
    ├── Environment 准备
    ├── Banner 打印
    ├── 创建 ApplicationContext
    ├── Context 准备
    ├── Context 刷新 ← 核心
    │    ├── 加载 BeanDefinition
    │    ├── 自动配置生效
    │    ├── 内嵌容器启动
    │    └── 组件扫描注册
    ├── Runner 执行
    └── StopWatch.stop()
```

---

## Q2：启动过程中发布哪些事件？顺序是什么？

**A**：

| 顺序 | 事件 | 说明 |
|------|------|------|
| 1 | ApplicationStartingEvent | 应用启动开始 |
| 2 | ApplicationEnvironmentPreparedEvent | Environment 准备完成 |
| 3 | ApplicationContextInitializedEvent | Context 初始化完成 |
| 4 | ApplicationPreparedEvent | Context 刷新前 |
| 5 | ContextRefreshedEvent | Context 刷新完成 |
| 6 | ServletWebServerInitializedEvent | Web 服务器初始化完成 |
| 7 | ApplicationStartedEvent | 应用启动完成 |
| 8 | ApplicationReadyEvent | 应用就绪 |
| - | ApplicationFailedEvent | **启动失败时** |

---

## Q3：如何监听 Spring Boot 启动事件？

**A**：有三种方式：

**方式一：@EventListener 注解**
```java
@Component
public class MyListener {
    
    @EventListener(ApplicationReadyEvent.class)
    public void onReady() {
        System.out.println("应用已就绪");
    }
    
    @EventListener(ApplicationFailedEvent.class)
    public void onFailed(ApplicationFailedEvent event) {
        System.out.println("启动失败：" + event.getException());
    }
}
```

**方式二：实现 ApplicationListener 接口**
```java
@Component
public class CustomListener implements ApplicationListener<ApplicationStartedEvent> {
    
    @Override
    public void onApplicationEvent(ApplicationStartedEvent event) {
        System.out.println("应用已启动");
    }
}
```

**方式三：通过 SpringApplication 添加**
```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication app = new SpringApplication(Application.class);
        app.addListeners(new CustomListener());
        app.run(args);
    }
}
```

---

## Q4：Bean 的初始化顺序是怎样的？

**A**：

```
1. BeanDefinition 加载
        ↓
2. BeanFactoryPostProcessor 执行
        ↓
3. Bean 实例化（构造函数）
        ↓
4. Bean 属性填充（@Autowired 注入）
        ↓
5. BeanPostProcessor.postProcessBeforeInitialization
        ↓
6. @PostConstruct 方法执行
        ↓
7. InitializingBean.afterPropertiesSet()
        ↓
8. BeanPostProcessor.postProcessAfterInitialization
        ↓
9. Bean 就绪
```

---

## Q5：CommandLineRunner 和 ApplicationRunner 的区别？

**A**：

| 对比 | CommandLineRunner | ApplicationRunner |
|------|-------------------|-------------------|
| **参数类型** | `String... args` | `ApplicationArguments args` |
| **解析能力** | 原始参数 | 可解析 `--key=value` |
| **使用方式** | 实现接口或 @Bean | 同左 |
| **执行顺序** | @Order 注解或 Ordered 接口 | 同左 |

**示例**：
```java
@Component
@Order(1)
public class MyCommandLineRunner implements CommandLineRunner {
    @Override
    public void run(String... args) throws Exception {
        System.out.println("原始参数：" + Arrays.toString(args));
    }
}

@Component
@Order(2)
public class MyApplicationRunner implements ApplicationRunner {
    @Override
    public void run(ApplicationArguments args) throws Exception {
        System.out.println("选项参数：" + args.getOptionNames());
        System.out.println("非选项参数：" + args.getNonOptionArgs());
    }
}
```

---

## Q6：如何在启动时获取 Environment？

**A**：

**方式一：监听事件**
```java
@Component
public class EnvListener implements ApplicationListener<ApplicationEnvironmentPreparedEvent> {
    @Override
    public void onApplicationEvent(ApplicationEnvironmentPreparedEvent event) {
        ConfigurableEnvironment env = event.getEnvironment();
        String port = env.getProperty("server.port", "8080");
        System.out.println("端口：" + port);
    }
}
```

**方式二：实现 EnvironmentAware**
```java
@Component
public class EnvAwareBean implements EnvironmentAware {
    private Environment env;
    
    @Override
    public void setEnvironment(Environment env) {
        this.env = env;
    }
    
    public void doSomething() {
        String name = env.getProperty("spring.application.name");
    }
}
```

**方式三：在 Runner 中获取**
```java
@Component
public class MyRunner implements CommandLineRunner {
    @Autowired
    private Environment env;
    
    @Override
    public void run(String... args) {
        String name = env.getProperty("spring.application.name");
    }
}
```

---

## Q7：Spring Boot 如何判断 Web 应用类型？

**A**：

```java
// 判断逻辑
if (存在 spring-webflux 类 && 不存在 spring-webmvc) {
    return REACTIVE;
}
if (存在 javax.servlet.Servlet || 
    存在 org.springframework.web.context.support.GenericWebApplicationContext) {
    return SERVLET;
}
return NONE;
```

**三种类型**：

| 类型 | 场景 | ApplicationContext |
|------|------|-------------------|
| SERVLET | Spring MVC 应用（默认） | AnnotationConfigServletWebServerApplicationContext |
| REACTIVE | WebFlux 响应式应用 | AnnotationConfigReactiveWebServerApplicationContext |
| NONE | 非 Web 应用 | AnnotationConfigApplicationContext |
