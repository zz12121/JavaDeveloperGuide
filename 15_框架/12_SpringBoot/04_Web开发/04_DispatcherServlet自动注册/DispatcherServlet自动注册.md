# DispatcherServlet自动注册

## 先说结论

SpringBoot 自动通过 `DispatcherServletAutoConfiguration` 注册 `DispatcherServlet`，映射路径为 `/`。**无需手动配置 web.xml**。

## 深度解析

### 核心概念

| 类 | 作用 |
|---|------|
| DispatcherServletAutoConfiguration | DispatcherServlet 自动配置 |
| DispatcherServletRegistrationBean | Servlet 注册 Bean |
| ServletRegistrationBean | Servlet 注册基类 |

### 自动注册原理

```java
// DispatcherServletAutoConfiguration
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE)
public class DispatcherServletAutoConfiguration {
    
    @Bean
    public DispatcherServlet dispatcherServlet() {
        return new DispatcherServlet();
    }
    
    @Bean
    public DispatcherServletRegistrationBean dispatcherServletRegistration(
            DispatcherServlet dispatcherServlet) {
        return new DispatcherServletRegistrationBean(dispatcherServlet, "/");
    }
}
```

### 映射路径

| 配置 | 映射路径 |
|------|----------|
| 默认 | `/` - 拦截所有请求 |
| 可配置 | `/*` - 包括 JSP（不推荐） |

### 自定义映射路径

```yaml
spring:
  mvc:
    servlet:
      path: /api/*
```

## 易错点/踩坑

- ❌ 设置 `spring.mvc.servlet.path=/api` → 只拦截 `/api` 下的请求
- ❌ 同时存在 `DispatcherServlet` + `spring.mvc.servlet.path` → 可能冲突
- ❌ 拦截所有请求但 Controller 只处理部分 → 其他请求返回 404

## 代码示例

### 自定义 DispatcherServlet

```java
@Configuration
public class WebMvcConfig {
    
    @Bean
    public DispatcherServlet dispatcherServlet() {
        DispatcherServlet servlet = new DispatcherServlet();
        servlet.setThrowExceptionIfNoHandlerFound(true);
        return servlet;
    }
    
    @Bean
    public DispatcherServletRegistrationBean dispatcherServletRegistration(
            DispatcherServlet dispatcherServlet) {
        return new DispatcherServletRegistrationBean(dispatcherServlet, "/api");
    }
}
```

## 图解/流程

```
应用启动
        ↓
DispatcherServletAutoConfiguration
        ↓
创建 DispatcherServlet 实例
        ↓
创建 DispatcherServletRegistrationBean
        ↓
设置映射路径为 "/"
        ↓
ServletContext 自动注册
        ↓
请求入口就绪
```

## 关联知识点

- `03_WebMvcAutoConfiguration自动配置原理`：自动配置整体机制
- `07_@RequestMapping请求映射`：DispatcherServlet 如何找到处理方法
