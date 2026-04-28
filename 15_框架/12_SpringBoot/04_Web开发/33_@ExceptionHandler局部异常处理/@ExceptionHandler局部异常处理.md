# @ExceptionHandler局部异常处理

## 先说结论

`@ExceptionHandler` 用于在单个 Controller 中处理异常，**优先级高于 @ControllerAdvice**。

## 深度解析

### 基本使用

```java
@Controller
@RequestMapping("/user")
public class UserController {
    
    @ExceptionHandler(BusinessException.class)
    @ResponseBody
    public Result handleBusinessException(BusinessException e) {
        return Result.error(e.getMessage());
    }
    
    @ExceptionHandler(Exception.class)
    @ResponseBody
    public Result handleException(Exception e) {
        return Result.error("系统错误");
    }
}
```

### 处理多种异常

```java
@ExceptionHandler({BusinessException.class, ValidationException.class})
@ResponseBody
public Result handleException(Exception e) {
    return Result.error(e.getMessage());
}
```

## 关联知识点

- `32_全局异常处理器`：全局异常处理
- `34_@ControllerAdvice全局异常处理`：@ControllerAdvice 详解
