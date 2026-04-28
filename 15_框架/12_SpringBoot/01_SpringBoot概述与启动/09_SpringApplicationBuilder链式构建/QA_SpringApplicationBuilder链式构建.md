# SpringApplicationBuilder 链式构建

## Q1：SpringApplicationBuilder 和 SpringApplication 有什么区别？

**A**：

| 对比 | SpringApplication | SpringApplicationBuilder |
|------|-------------------|--------------------------|
| **API 风格** | setter 方法 | Fluent API（链式调用） |
| **父子容器** | 不支持 | 支持 |
| **多容器** | 单容器 | 多容器（父子关系） |
| **使用场景** | 简单场景 | 复杂企业应用 |

**Builder 本质**：Builder 内部也是创建 SpringApplication

```java
// SpringApplicationBuilder 源码简化
public class SpringApplicationBuilder {
    private final SpringApplicationBuilder parent;
    private final SpringApplication springApplication;
    
    public SpringApplicationBuilder(Class<?>... sources) {
        this.springApplication = new SpringApplication(sources);
    }
    
    public SpringApplicationBuilder bannerMode(Banner.Mode mode) {
        this.springApplication.setBannerMode(mode);
        return this;
    }
    
    public ConfigurableApplicationContext run(String... args) {
        // 处理父子容器逻辑
        return this.springApplication.run(args);
    }
}
```

---

## Q2：什么时候需要使用父子容器？

**A**：父子容器适用于**微服务架构**和**模块化开发**：

| 场景 | 说明 |
|------|------|
| **基础设施共享** | 父容器放公共组件（DataSource、Redis），子容器放业务模块 |
| **模块隔离** | 不同子容器管理不同业务域 |
| **Spring Security 多模块** | 不同子应用可能有不同的安全配置 |

**示例 - 微服务基础库**：
```java
// 父容器：基础服务
public class InfrastructureConfig {
    @Bean
    public DataSource dataSource() {
        return DruidDataSource.create(...);
    }
}

// 子容器：业务服务
public class BusinessConfig {
    @Bean
    public UserService userService(DataSource dataSource) {
        // 可直接使用父容器的 DataSource
        return new UserService(dataSource);
    }
}

// 启动
new SpringApplicationBuilder(InfrastructureConfig.class)
    .web(WebApplicationType.NONE)
    .run(args);  // 父容器

new SpringApplicationBuilder(BusinessConfig.class)
    .parent(parentContext)
    .web(WebApplicationType.SERVLET)
    .run(args);  // 子容器
```

---

## Q3：父子容器的 Bean 访问规则是什么？

**A**：

```
子容器可以访问父容器的 Bean
父容器不能访问子容器的 Bean
```

```java
// 父容器 Bean
@Bean
public ParentService parentService() {
    return new ParentService();
}

// 子容器 Bean
@Bean
public ChildService childService() {
    return new ChildService();
}

// 测试
ApplicationContext child = getChildContext();

// ✓ 子容器可以获取自己的 Bean
child.getBean(ChildService.class);

// ✓ 子容器可以获取父容器的 Bean
child.getBean(ParentService.class);

// ✗ 父容器不能获取子容器的 Bean
parent.getBean(ChildService.class); // 抛出异常
```

---

## Q4：Builder 的 profiles() 和 profiles(String...) 区别？

**A**：**没有区别**，后者只是可变参数版本：

```java
// 以下两种写法等价
builder.profiles("dev")
builder.profiles("dev", "prod")

// 内部实现
public SpringApplicationBuilder profiles(String... profiles) {
    this.springApplication.setAdditionalProfiles(profiles);
    return this;
}
```

---

## Q5：如何在 Builder 中设置 properties？

**A**：有多种方式：

```java
new SpringApplicationBuilder(App.class)
    // 方式一：可变参数
    .properties("server.port=8080", "server.servlet.context-path=/demo")
    
    // 方式二：Map
    .properties(Map.of(
        "server.port", "8080",
        "spring.application.name", "demo"
    ))
    
    // 方式三：字符串（带连接符）
    .properties("server.port:" + 8080)  // 注意是冒号
    .properties("server.port=" + 8080)  // 或等号
```

---

## Q6：Spring Boot 2.4+ 的新写法？

**A**：Spring Boot 2.4 简化了配置方式：

```java
// 2.4 之前
new SpringApplicationBuilder(App.class)
    .bannerMode(Banner.Mode.OFF)
    .run(args);

// 2.4+ 更简洁
SpringApplication.run(App.class, args);  // 直接用静态方法即可

// 如需自定义
SpringApplication app = new SpringApplication(App.class);
app.setBannerMode(Banner.Mode.OFF);
// 等价于 Builder 写法
```

**但 Builder 仍有用武之地**：
```java
// 多容器场景必须用 Builder
new SpringApplicationBuilder(ChildConfig.class)
    .parent(parentContext)
    .run(args);

// profiles 激活
new SpringApplicationBuilder(App.class)
    .profiles("dev", "prod")
    .run(args);
```
