# SpringApplication 静态方法

## 先说结论

`SpringApplication` 提供两种启动方式：
- `run(Class, args)`：静态方法，最常用
- `create(args)` + `run(args)`：实例方法，可自定义配置

## 深度解析

### API 对比

| 方法 | 类型 | 灵活性 | 使用场景 |
|------|------|--------|----------|
| `run(Class, args)` | 静态 | 低 | 标准启动 |
| `create(args).run(args)` | 实例 | 高 | 需要自定义配置 |

### run 静态方法

```java
// 源码
public static ConfigurableApplicationContext run(Class<?> primarySource, String... args) {
    return new SpringApplication(primarySource).run(args);
}
```

### create 实例方法

```java
// 创建实例
SpringApplication app = SpringApplication.create(args);

// 自定义配置
app.setBannerMode(Banner.Mode.OFF);
app.setAddWebListener(true);

// 启动
ConfigurableApplicationContext ctx = app.run(args);
```

### 自定义配置项

| 配置项 | 方法 | 说明 |
|--------|------|------|
| Banner 模式 | `setBannerMode()` | OFF / CONSOLE / LOG |
| 关闭 Web 类型 | `setWebApplicationType()` | SERVLET / REACTIVE / NONE |
| 添加初始化器 | `addInitializers()` | ApplicationContextInitializer |
| 添加监听器 | `addListeners()` | ApplicationListener |
| 设置主配置类 | `setSources()` | Set<String> |
| 注册 shutdown hook | `setRegisterShutdownHook()` | boolean |

## 易错点/踩坑

- ❌ 混淆静态 run 和实例 run 的用法
- ❌ 在 main 方法执行前调用 Spring 容器
- ❌ 自定义配置后忘记调用 run()

## 代码示例

### 标准启动

```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

### 自定义启动

```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        // 创建 SpringApplication
        SpringApplication app = new SpringApplication(Application.class);
        
        // 自定义 Banner
        app.setBannerMode(Banner.Mode.OFF);
        
        // 添加监听器
        app.addListeners(new MyListener());
        
        // 添加初始化器
        app.addInitializers(new MyInitializer());
        
        // 设置 Web 类型
        // app.setWebApplicationType(WebApplicationType.NONE);
        
        // 启动
        ConfigurableApplicationContext ctx = app.run(args);
        
        // 可以从 ctx 中获取 Bean
        MyService myService = ctx.getBean(MyService.class);
    }
}
```

### Fluent API（Spring Boot 2.4+）

```java
public static void main(String[] args) {
    new SpringApplicationBuilder(DemoApplication.class)
        .bannerMode(Banner.Mode.OFF)
        .run(args);
}
```

## 关联知识点

- `SpringApplication.run()` 启动流程
- `SpringApplicationBuilder` 链式构建
- Banner 自定义
- SpringApplication 事件
