# SpringApplicationBuilder 链式构建

## 先说结论

`SpringApplicationBuilder` 提供 Fluent API 链式调用方式配置 SpringApplication，支持父子容器、环境profiles、资源加载等高级特性。

## 深度解析

### 基本用法

```java
new SpringApplicationBuilder(DemoApplication.class)
    .bannerMode(Banner.Mode.OFF)
    .web(WebApplicationType.SERVLET)
    .run(args);
```

### Builder 方法一览

| 方法 | 说明 | 示例 |
|------|------|------|
| `sources(Class...)` | 设置主配置类 | `sources(Application.class)` |
| `banner(Banner)` | 自定义 Banner | `banner(new MyBanner())` |
| `bannerMode(Banner.Mode)` | Banner 模式 | `bannerMode(Banner.Mode.OFF)` |
| `web(WebApplicationType)` | Web 类型 | `web(WebApplicationType.NONE)` |
| `applicationName(String)` | 应用名称 | `applicationName("myapp")` |
| `profiles(String...)` | 激活 profiles | `profiles("dev", "prod")` |
| `properties(Map)` | 设置属性 | `properties("server.port=8080")` |
| `initializers(ApplicationContextInitializer...)` | 添加初始化器 | `initializers(new MyInit())` |
| `listeners(ApplicationListener...)` | 添加监听器 | `listeners(new MyListener())` |
| `parent(Class)` | 父子容器 | `parent(ParentConfig.class)` |
| `child(Class)` | 子容器配置 | `child(ChildConfig.class)` |

### 父子容器

```
┌─────────────────────────────────────────────────────────────┐
│                      父子容器架构                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   ┌─────────────────┐                                       │
│   │   Parent App    │ ← 基础设施 Bean                       │
│   │ ApplicationContext                                      │
│   └────────┬────────┘                                       │
│            │ child()                                        │
│            ↓                                                 │
│   ┌─────────────────┐                                       │
│   │   Child App     │ ← 业务 Bean（可访问 Parent）           │
│   │ ApplicationContext                                      │
│   └─────────────────┘                                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### properties vs application.yml

```java
// 方式一：Builder properties
new SpringApplicationBuilder(App.class)
    .properties("server.port=9090")
    .properties("spring.datasource.url=jdbc:mysql://localhost:3306/test")
    .run(args);

// 方式二：application.yml
// server.port=9090
// spring.datasource.url=jdbc:mysql://localhost:3306/test
```

## 易错点/踩坑

- ❌ 父子容器导致 Bean 查找混乱
- ❌ properties 格式写错
- ❌ 不理解父子容器的 Bean 访问规则

## 代码示例

### 标准链式调用

```java
public static void main(String[] args) {
    new SpringApplicationBuilder(DemoApplication.class)
        .bannerMode(Banner.Mode.OFF)
        .web(WebApplicationType.SERVLET)
        .profiles("dev")
        .properties(
            "server.port=8080",
            "spring.application.name=demo"
        )
        .run(args);
}
```

### 父子容器示例

```java
public static void main(String[] args) {
    // 父容器：基础设施
    ConfigurableApplicationContext parent = new SpringApplicationBuilder(ParentConfig.class)
        .web(WebApplicationType.NONE)
        .run(args);
    
    // 子容器：业务应用
    ConfigurableApplicationContext child = new SpringApplicationBuilder(ChildConfig.class)
        .parent(parent)
        .web(WebApplicationType.SERVLET)
        .run(args);
    
    // 子容器可以访问父容器的 Bean
    Object bean = child.getBean("parentBean");
}
```

## 关联知识点

- SpringApplication 静态方法
- SpringApplication.run() 启动流程
- Banner 自定义
- profiles 环境配置
