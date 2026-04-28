# 自定义ErrorAttributes

## 先说结论

通过实现 `ErrorController` 或自定义 `ErrorAttributes`，可以完全控制错误响应的格式和内容。

## 深度解析

### 方式1：自定义 ErrorAttributes

```java
@Component
public class CustomErrorAttributes implements ErrorAttributes {
    
    @Override
    public Map<String, Object> getErrorAttributes(
            ErrorAttributesRequestScope requestAttributes) {
        Map<String, Object> errorAttributes = new HashMap<>();
        errorAttributes.put("code", 500);
        errorAttributes.put("msg", "系统错误");
        return errorAttributes;
    }
}
```

### 方式2：自定义 ErrorController

```java
@ControllerAdvice
public class CustomErrorController implements ErrorController {
    
    @RequestMapping("/error")
    public Result handleError(HttpServletRequest request) {
        Integer status = (Integer) request.getAttribute("jakarta.servlet.error.status_code");
        String message = (String) request.getAttribute("jakarta.servlet.error.message");
        return Result.error(status, message);
    }
}
```

### Spring Boot 2.x vs 3.x

```java
// Spring Boot 2.x
request.getAttribute("javax.servlet.error.status_code");

// Spring Boot 3.x
request.getAttribute("jakarta.servlet.error.status_code");
```

## 易错点/踩坑

- ❌ Spring Boot 2.x 和 3.x 的属性名不同
- ❌ 自定义 ErrorController 后默认页面失效
- ❌ ErrorAttributes 返回 null → 可能导致响应异常

## 关联知识点

- `32_全局异常处理器`：异常处理基础
- `34_@ControllerAdvice全局异常处理`：@ControllerAdvice 详解
