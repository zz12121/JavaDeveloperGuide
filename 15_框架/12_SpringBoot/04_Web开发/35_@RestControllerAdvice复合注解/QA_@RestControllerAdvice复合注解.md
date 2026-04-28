# @RestControllerAdvice复合注解 - QA

## Q1：@RestControllerAdvice 内部怎么实现的？

**A**：`@RestControllerAdvice` 本质上是一个组合注解。

```java
@RestControllerAdvice
public @interface RestControllerAdvice {
    // 等价于
    @ControllerAdvice
    @ResponseBody
}
```

---

## Q2：@RestControllerAdvice 能返回视图吗？

**A**：不能。直接返回 JSON。

**如需返回视图**：使用 `@ControllerAdvice` 并显式返回视图名。

---

## Q3：最佳实践是什么？

**A**：

```java
// 统一异常处理
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    // 业务异常
    @ExceptionHandler(BusinessException.class)
    public Result handleBusiness(BusinessException e) {}
    
    // 参数校验异常
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public Result handleValidation(MethodArgumentNotValidException e) {}
    
    // 系统异常
    @ExceptionHandler(Exception.class)
    public Result handleException(Exception e) {}
}
```
