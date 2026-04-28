# 全局异常处理器 - QA

## Q1：@ControllerAdvice 和 @RestControllerAdvice 区别？

**A**：

| 对比 | @ControllerAdvice | @RestControllerAdvice |
|------|-------------------|----------------------|
| 返回值处理 | 视图解析 | @ResponseBody 自动添加 |
| 适用场景 | 混合项目 | RESTful API |

**@RestControllerAdvice = @ControllerAdvice + @ResponseBody**

---

## Q2：异常处理优先级？

**A**：更具体的异常优先。

```java
@RestControllerAdvice
public class Handler {
    
    @ExceptionHandler(NullPointerException.class)  // 先匹配
    public Result handleNPE(NullPointerException e) {}
    
    @ExceptionHandler(RuntimeException.class)  // 后匹配
    public Result handleRuntime(RuntimeException e) {}
    
    @ExceptionHandler(Exception.class)  // 最后匹配
    public Result handleException(Exception e) {}
}
```

---

## Q3：如何获取原始异常信息？

**A**：

```java
@ExceptionHandler(Exception.class)
public Result handleException(Exception e, HandlerParameter parameter) {
    e.getCause();           // 根因
    e.getStackTrace();      // 堆栈
    return Result.error(e.getMessage());
}
```

---

## Q4：自定义错误页面？

**A**：

```
resources/
├── static/
│   └── error/
│       ├── 404.html
│       ├── 500.html
│       └── error.html
```
