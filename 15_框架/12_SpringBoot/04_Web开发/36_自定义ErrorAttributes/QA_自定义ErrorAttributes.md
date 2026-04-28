# 自定义ErrorAttributes - QA

## Q1：ErrorAttributes 和 ErrorController 区别？

**A**：

| 对比 | ErrorAttributes | ErrorController |
|------|-----------------|-----------------|
| 粒度 | 只自定义错误属性 | 完全控制错误处理 |
| 使用方式 | 实现接口 | 实现接口 |
| 灵活性 | 较低 | 高 |

---

## Q2：Spring Boot 3.x 属性名变了？

**A**：是的。

```java
// Spring Boot 2.x
request.getAttribute("javax.servlet.error.status_code");

// Spring Boot 3.x
request.getAttribute("jakarta.servlet.error.status_code");
```

---

## Q3：如何返回统一格式错误？

**A**：

```java
@ControllerAdvice
public class CustomErrorController implements ErrorController {
    
    @RequestMapping("/error")
    public Result handleError(HttpServletRequest request) {
        Integer status = (Integer) request.getAttribute(
            "jakarta.servlet.error.status_code");
        
        Map<String, Object> error = new HashMap<>();
        error.put("code", status);
        error.put("message", getMessage(request));
        
        return Result.error(error);
    }
}
```

---

## Q4：@ControllerAdvice 中的 @ExceptionHandler 和 ErrorController 哪个优先？

**A**：@ExceptionHandler 优先。

**顺序**：
1. @ExceptionHandler（@ControllerAdvice）
2. ErrorController（兜底）

---

## Q5：如何获取异常对象？

**A**：

```java
@Override
public Map<String, Object> getErrorAttributes(
        ErrorAttributesRequestScope requestAttributes) {
    Throwable exception = requestAttributes.getError();
    return Map.of("message", exception.getMessage());
}
```
