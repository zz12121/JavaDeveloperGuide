# @RestControllerAdvice复合注解

## 先说结论

`@RestControllerAdvice` = `@ControllerAdvice` + `@ResponseBody`，是 RESTful API 全局异常处理的首选。

## 深度解析

### 注解源码

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@ControllerAdvice
@ResponseBody
public @interface RestControllerAdvice {
    @AliasFor(annotation = ControllerAdvice.class)
    String[] basePackages() default {};
}
```

### 与 @ControllerAdvice 的区别

| 对比 | @ControllerAdvice | @RestControllerAdvice |
|------|-------------------|----------------------|
| @ResponseBody | 需要显式添加 | 自动添加 |
| 返回视图 | 支持 | 不支持 |
| 返回 JSON | 需要显式添加 | 自动添加 |
| 推荐场景 | 混合项目 | RESTful API |

### 使用场景

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(BusinessException.class)
    public Result handleBusiness(BusinessException e) {
        return Result.error(e.getCode(), e.getMessage());
    }
    
    @ExceptionHandler(Exception.class)
    public Result handleException(Exception e) {
        return Result.error("系统错误");
    }
}
```

## 关联知识点

- `34_@ControllerAdvice全局异常处理`：@ControllerAdvice 详解
- `32_全局异常处理器`：异常处理基础
