# DispatcherServlet自动注册 - QA

## Q1：DispatcherServlet 的作用是什么？

**A**：前端控制器，是 Spring MVC 的核心。

**职责**：
1. 接收所有请求
2. 根据 URL 找到对应的 Handler（Controller 方法）
3. 调用 Handler 处理请求
4. 返回 ModelAndView
5. 渲染视图，返回响应

**流程**：
```
请求 → DispatcherServlet → HandlerMapping（找Controller）
                    ↓
            HandlerAdapter（执行Controller）
                    ↓
            返回 ModelAndView
                    ↓
            ViewResolver（解析视图）
                    ↓
            View（渲染页面）
                    ↓
            响应
```

---

## Q2：如何修改 DispatcherServlet 的映射路径？

**A**：两种方式：

```yaml
# 方式1：配置文件
spring:
  mvc:
    servlet:
      path: /api

# 方式2：代码注册
@Configuration
public class WebMvcConfig {
    @Bean
    public DispatcherServletRegistrationBean registration() {
        DispatcherServlet servlet = new DispatcherServlet();
        return new DispatcherServletRegistrationBean(servlet, "/api");
    }
}
```

---

## Q3：可以注册多个 DispatcherServlet 吗？

**A**：可以。

**场景**：微服务中不同模块需要不同的 Servlet 配置。

```java
@Configuration
public class MultiDispatcherConfig {
    
    @Bean
    public DispatcherServletRegistrationBean webDispatcher() {
        DispatcherServlet servlet = new DispatcherServlet();
        servlet.setContextConfigLocation("classpath:web-context.xml");
        return new DispatcherServletRegistrationBean(servlet, "/web/*");
    }
    
    @Bean
    public DispatcherServletRegistrationBean apiDispatcher() {
        DispatcherServlet servlet = new DispatcherServlet();
        servlet.setContextConfigLocation("classpath:api-context.xml");
        return new DispatcherServletRegistrationBean(servlet, "/api/*");
    }
}
```

---

## Q4：为什么 Controller 返回 404？

**A**：常见原因：

| 原因 | 解决方案 |
|------|----------|
| DispatcherServlet 映射路径问题 | 检查路径配置 |
| @RequestMapping 路径错误 | 检查注解路径 |
| 组件扫描未包含 | 检查 @ComponentScan |
| 请求方法不匹配 | GET/POST 方法对应 |
| 静态资源拦截 | 排除静态资源路径 |

---

## Q5：SpringBoot 2.x 和 1.x 注册方式有什么区别？

**A**：

| 版本 | 注册方式 |
|------|----------|
| 1.x | `EmbeddedServletContainerCustomizer` |
| 2.x | `DispatcherServletRegistrationBean` |

```java
// SpringBoot 2.x 方式
@Bean
public DispatcherServletRegistrationBean registration() {
    return new DispatcherServletRegistrationBean(dispatcherServlet(), "/");
}
```
