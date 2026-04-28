# @ControllerAdvice全局异常处理

## 先说结论

`@ControllerAdvice` 是 Spring 3.2 引入的注解，用于全局处理 Controller 的异常，支持所有 Controller。

## 深度解析

### 注解源码

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface ControllerAdvice {
    @AliasFor("basePackages")
    String[] value() default {};
    
    @AliasFor("value")
    String[] basePackages() default {};
    
    Class<?>[] basePackageClasses() default {};
    
    Class<?>[] assignableTypes() default {};
    
    String[] annotations() default {};
}
```

### 作用范围

```java
// 全局生效
@ControllerAdvice
public class GlobalHandler {}

// 指定包
@ControllerAdvice(basePackages = "com.example.controller")

// 指定类
@ControllerAdvice(assignableTypes = {UserController.class, OrderController.class})

// 指定注解
@ControllerAdvice(annotations = @RestController)
```

### 完整示例

```java
@RestControllerAdvice(basePackages = "com.example.api")
public class ApiExceptionHandler {
    
    @ExceptionHandler(BusinessException.class)
    public Result handleBusiness(BusinessException e) {
        return Result.error(e.getCode(), e.getMessage());
    }
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public Result handleValidation(MethodArgumentNotValidException e) {
        String message = e.getBindingResult().getFieldErrors().stream()
                .map(FieldError::getDefaultMessage)
                .collect(Collectors.joining(", "));
        return Result.error(400, message);
    }
}
```

## 关联知识点

- `32_全局异常处理器`：异常处理基础
- `33_@ExceptionHandler局部异常处理`：局部异常处理
