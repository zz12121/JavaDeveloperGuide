# 视图解析器InternalResourceViewResolver

## 先说结论

`InternalResourceViewResolver` 是 JSP 视图的解析器，将逻辑视图名解析为 JSP 文件路径。**Spring Boot 已自动配置**。

## 深度解析

### 自动配置

```yaml
spring:
  mvc:
    view:
      prefix: /WEB-INF/views/   # 视图前缀
      suffix: .jsp              # 视图后缀
```

### 自定义配置

```java
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {
    @Override
    public void configureViewResolvers(ViewResolverRegistry registry) {
        registry.jsp("/WEB-INF/views/", ".jsp");
    }
}
```

### 视图解析流程

```
Controller 返回: "user/list"
        ↓
ViewResolver 解析
        ↓
前缀 + 逻辑视图名 + 后缀
        ↓
/WEB-INF/views/user/list.jsp
```

## 关联知识点

- `24_JSP与JSTL支持`：JSP 模板引擎
- `20_Thymeleaf模板引擎`：Thymeleaf 模板引擎
