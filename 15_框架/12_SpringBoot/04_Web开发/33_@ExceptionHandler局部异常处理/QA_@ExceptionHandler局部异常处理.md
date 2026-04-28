# @ExceptionHandler局部异常处理 - QA

## Q1：局部和全局异常处理哪个优先？

**A**：局部优先。

**顺序**：
1. @ExceptionHandler（Controller 局部）
2. @ExceptionHandler（@ControllerAdvice 全局）
3. Spring Boot 默认异常处理

---

## Q2：参数能获取什么信息？

**A**：

```java
@ExceptionHandler(Exception.class)
public Result handleException(
    Exception e,                      // 异常对象
    HttpServletRequest request,       // 请求信息
    HttpServletResponse response,     // 响应对象
    HandlerMethod handlerMethod       // 处理方法
) {
    return Result.error(e.getMessage());
}
```

---

## Q3：能返回视图吗？

**A**：可以。

```java
@ExceptionHandler(Exception.class)
public String handleException(Exception e, Model model) {
    model.addAttribute("error", e.getMessage());
    return "error/500";
}
```
