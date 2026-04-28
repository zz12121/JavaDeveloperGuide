# @ControllerAdvice全局异常处理 - QA

## Q1：@ControllerAdvice 能拦截哪些异常？

**A**：

| 异常类型 | 说明 |
|----------|------|
| Controller 抛出的异常 | ✅ 拦截 |
| @ExceptionHandler 抛出的异常 | ❌ 不拦截 |
| Filter/Interceptor 异常 | ❌ 不拦截 |
| 渲染视图时的异常 | ✅ 拦截 |

---

## Q2：如何区分不同环境返回不同格式？

**A**：

```java
@ControllerAdvice
public class GlobalHandler {
    
    @ExceptionHandler(Exception.class)
    public Object handleException(Exception e, HttpServletRequest request) {
        String accept = request.getHeader("Accept");
        
        if (accept != null && accept.contains("application/json")) {
            return Result.error(e.getMessage());
        }
        return "error/500";
    }
}
```

---

## Q3：@RestControllerAdvice 和 @ControllerAdvice + @ResponseBody 区别？

**A**：`@RestControllerAdvice` 是简化写法，行为相同。

```java
// 等价
@RestControllerAdvice
public class Handler1 {}

@ControllerAdvice
@ResponseBody
public class Handler2 {}
```

---

## Q4：多个 @ControllerAdvice 优先级？

**A**：按 @Order 注解或实现 Ordered 接口。

```java
@Order(1)
@ControllerAdvice
public class Handler1 {}

@Order(2)
@ControllerAdvice
public class Handler2 {}
```
