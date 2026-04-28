# SpringApplication.run() 启动流程

## 先说结论

`SpringApplication.run()` 完成了从创建应用上下文到启动内嵌容器的完整流程，核心步骤包括：准备环境、创建上下文、刷新上下文、启动容器。

## 深度解析

### 启动流程概览

```
┌─────────────────────────────────────────────────────────────────┐
│                    SpringApplication.run() 流程                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  SpringApplication.run(Application.class, args)                   │
│         │                                                        │
│         ├── 1. 创建 SpringApplication 实例                      │
│         │        │                                               │
│         │        ├── 判断应用类型（REACTIVE/SERVLET/NONE）       │
│         │        ├── 加载 spring.factories                       │
│         │        └── 初始化初始构造器                             │
│         │                                                        │
│         ├── 2. 执行 run() 方法                                   │
│         │        │                                               │
│         │        ├── StopWatch 计时开始                          │
│         │        │                                               │
│         │        ├── BootstrapContext 创建                       │
│         │        │                                               │
│         │        ├── HeadlessProperty 赋值                       │
│         │        │                                               │
│         │        ├── SpringApplicationRunListeners 创建          │
│         │        │                                               │
│         │        ├── Environment 准备（ApplicationEnvironment）   │
│         │        │        │                                      │
│         │        │        ├── 监听器发布 EnvironmentPreparedEvent │
│         │        │        └── 绑定 SpringApplication              │
│         │        │                                               │
│         │        ├── Banner 打印                                 │
│         │        │                                               │
│         │        ├── 创建 ApplicationContext                     │
│         │        │        │                                      │
│         │        │        ├── AnnotationConfigServletWebServerApplicationContext│
│         │        │        └── ...                                │
│         │        │                                               │
│         │        ├── Context 准备                                │
│         │        │        │                                      │
│         │        │        ├── 监听器发布 ContextPreparedEvent     │
│         │        │        └── 加载/刷新 PropertySource            │
│         │        │                                               │
│         │        ├── Context 刷新                                │
│         │        │        │                                      │
│         │        │        ├── 激活 @PropertySource               │
│         │        │        ├── 激活 @PropertySourceConditions    │
│         │        │        ├── 加载 BeanDefinition                │
│         │        │        ├── BeanFactory 后置处理               │
│         │        │        ├── 注册 BeanFactory 后置处理器        │
│         │        │        ├── 激活 @ComponentScan               │
│         │        │        ├── 激活 @Configuration               │
│         │        │        ├── 注册 LoadTimeWeaver               │
│         │        │        ├── BeanFactory 初始化                │
│         │        │        ├── 激活 Bean 实例化后置处理器         │
│         │        │        ├── 激活 Bean 实例化                  │
│         │        │        ├── 激活 Bean 初始化后置处理器         │
│         │        │        ├── 激活 Bean 初始化                  │
│         │        │        ├── 激活 Bean 后置处理器               │
│         │        │        ├── 初始化内嵌容器                     │
│         │        │        └── 激活 @ComponentScan               │
│         │        │                                               │
│         │        ├── 监听器发布 ContextRefreshedEvent           │
│         │        │                                               │
│         │        ├── Runner 执行                                 │
│         │        │        │                                      │
│         │        │        ├── CommandLineRunner                  │
│         │        │        └── ApplicationRunner                 │
│         │        │                                               │
│         │        └── StopWatch 计时结束                         │
│         │                                                        │
│         └── 返回 ApplicationContext                              │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### 关键步骤详解

#### 1. 判断应用类型

```java
// SpringApplication.java
private WebApplicationType deduceWebApplicationType() {
    if (ClassUtils.isPresent("org.springframework.web.reactive.DispatcherHandler", ...)
            && !ClassUtils.isPresent("org.springframework.web.servlet.DispatcherServlet", ...)) {
        return WebApplicationType.REACTIVE;
    }
    for (String className : SERVLET_IGNORED) {
        if (ClassUtils.isPresent(className, ...)) {
            return WebApplicationType.NONE;
        }
    }
    return WebApplicationType.SERVLET;
}
```

#### 2. 加载 spring.factories

```
META-INF/spring.factories
    │
    └── ApplicationContextInitializer
    └── ApplicationListener
    └── SpringApplicationRunListener
    └── AutoConfigurationImportListener
    └── AutoConfigurationImportFilter
```

#### 3. 监听器事件顺序

| 事件 | 时机 |
|------|------|
| ApplicationStartingEvent | 启动时，Environment/Context 创建前 |
| ApplicationEnvironmentPreparedEvent | Environment 准备完成 |
| ApplicationContextInitializedEvent | Context 初始化完成，Bean 定义加载前 |
| ApplicationPreparedEvent | Context 刷新前 |
| ContextRefreshedEvent | Context 刷新完成 |
| ApplicationStartedEvent | 容器启动完成，Runner 执行前 |
| ApplicationReadyEvent | 应用就绪 |
| ApplicationFailedEvent | 启动失败 |

## 易错点/踩坑

- ❌ 不理解事件监听器的作用时机
- ❌ 在错误的事件中访问未初始化的 Bean
- ❌ 混淆 Runner 和 Bean 初始化的执行顺序

## 代码示例

### 监听启动事件

```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication app = new SpringApplication(Application.class);
        
        // 添加监听器
        app.addListeners(new MyApplicationListener());
        
        app.run(args);
    }
}

public class MyApplicationListener 
        implements ApplicationListener<ApplicationEnvironmentPreparedEvent> {
    
    @Override
    public void onApplicationEvent(ApplicationEnvironmentPreparedEvent event) {
        // Environment 已准备好，可以修改
        ConfigurableEnvironment env = event.getEnvironment();
        env.setDefaultProfiles("dev");
    }
}
```

### 使用 Runner 执行初始化

```java
@Component
public class DataInitializer implements CommandLineRunner {
    
    @Override
    public void run(String... args) throws Exception {
        // 在应用启动后执行
        System.out.println("应用启动完成，执行初始化");
    }
}
```

## 关联知识点

- `@SpringBootApplication` 组合注解
- SpringApplication 事件体系
- ApplicationRunner / CommandLineRunner
- 自动配置原理
