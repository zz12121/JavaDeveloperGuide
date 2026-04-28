# SpringBoot自动配置WebMvc

## 先说结论

SpringBoot 通过 `WebMvcAutoConfiguration` 自动配置好了 Web 开发所需的一切：DispatcherServlet、静态资源处理、视图解析器等。**大多数情况下零配置即可运行**。

## 深度解析

### 核心概念

| 组件 | 说明 |
|------|------|
| WebMvcAutoConfiguration | Web MVC 自动配置类 |
| WebMvcProperties | 配置属性类，绑定 `spring.mvc.*` |
| WebMvcAutoConfigurationAdapter | 适配器，提供可选配置钩子 |
| @EnableWebMvc | 用户添加此注解则禁用自动配置 |

### 自动配置内容

```
WebMvcAutoConfiguration 配置了：
├── DispatcherServletRegistrationBean    # DispatcherServlet 注册
├── RequestMappingHandlerMapping         # @RequestMapping 映射
├── RequestMappingHandlerAdapter         # 请求处理适配器
├── HttpMessageConverterBundle           # 消息转换器（JSON/XML）
├── ErrorMvcAutoConfiguration            # 错误页面自动配置
└── WelcomePageHandlerMapping            # 欢迎页
```

### @EnableWebMvc 的作用

添加 `@EnableWebMvc` 注解会导入 `DelegatingWebMvcConfiguration`，**完全接管 WebMvc 配置**，自动配置失效。

```java
// 禁用自动配置，自定义完全控制
@SpringBootApplication
@EnableWebMvc
public class App {
    public static void main(String[] args) {
        SpringApplication.run(App.class, args);
    }
}
```

## 易错点/踩坑

- ❌ 同时使用 `@EnableWebMvc` + 想用自动配置 → 自动配置被禁用
- ❌ 自定义 `WebMvcConfigurer` 后某些功能消失 → 检查是否覆盖了不该覆盖的方法
- ❌ 静态资源404 → 检查是否添加了 `@EnableWebMvc`

## 代码示例

### 自定义 WebMvcConfigurer（不禁用自动配置）

```java
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {
    
    @Override
    public void addViewControllers(ViewControllerRegistry registry) {
        // 自定义视图控制器，不影响其他自动配置
        registry.addViewController("/").setViewName("home");
    }
    
    @Override
    public void configureContentNegotiation(ContentNegotiationConfigurer configurer) {
        // 自定义内容协商
        configurer.favorParameter(true);
    }
}
```

## 图解/流程

```
应用启动
    ↓
WebMvcAutoConfiguration 自动配置
    ↓
检查是否存在 @EnableWebMvc
    ├── 是 → DelegatingWebMvcConfiguration 接管，禁用自动配置
    └── 否 → 自动配置生效，用户配置 + 自动配置合并
    ↓
DispatcherServlet 注册到容器
    ↓
请求入口就绪
```

## 关联知识点

- `02_@SpringBootApplication注解三合一`：@EnableAutoConfiguration 触发自动配置
- `03_WebMvcAutoConfiguration自动配置原理`：深入原理
