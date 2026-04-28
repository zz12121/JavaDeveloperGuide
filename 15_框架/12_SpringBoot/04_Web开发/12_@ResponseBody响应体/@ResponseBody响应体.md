# @ResponseBody响应体

## 先说结论

`@ResponseBody` 将方法返回值写入 HTTP 响应体，配合 HttpMessageConverter 自动序列化为 JSON/XML 等格式。**是 RESTful API 返回数据的标准方式**。

## 深度解析

### 工作原理

```
方法返回: User{name="张三", age=25}
        ↓
HttpMessageConverter（Jackson）
        ↓
序列化为 JSON
        ↓
HTTP 响应体
Content-Type: application/json

{"name": "张三", "age": 25}
```

### 基本使用

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@ResponseBody
public @interface ResponseBody {}
```

### 与 @RestController 的关系

```java
// @RestController = @Controller + @ResponseBody
@RestController
public class UserController {
    // 所有方法自动添加 @ResponseBody
}

// 等价于
@Controller
@ResponseBody
public class UserController {
    // 需要显式添加 @ResponseBody
}
```

### 返回类型

| 返回类型 | 行为 |
|----------|------|
| String | 直接返回文本 |
| Object | 序列化为 JSON |
| ResponseEntity | 自定义响应状态码和头 |
| void | 不返回内容 |

## 易错点/踩坑

- ❌ @Controller + @ResponseBody → 方法返回的是视图名，不是响应体
- ❌ 返回 null → 响应体为空，但状态码可能是 200
- ❌ 中文乱码 → 配置 produces 或全局编码

## 代码示例

### 设置响应类型

```java
@GetMapping(value = "/user/{id}", produces = "application/json;charset=UTF-8")
@ResponseBody
public User getUser(@PathVariable Long id) {
    return userService.getById(id);
}
```

### 返回 JSON Result

```java
// 统一响应结构
public class Result<T> {
    private int code;
    private String message;
    private T data;
    
    public static <T> Result<T> success(T data) {
        Result<T> result = new Result<>();
        result.setCode(200);
        result.setMessage("success");
        result.setData(data);
        return result;
    }
}

// 使用
@GetMapping("/user/{id}")
@ResponseBody
public Result<User> getUser(@PathVariable Long id) {
    return Result.success(userService.getById(id));
}
```

### 全局配置

```java
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {
    @Override
    public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
        StringHttpMessageConverter stringConverter = new StringHttpMessageConverter(StandardCharsets.UTF_8);
        converters.add(0, stringConverter);
    }
}
```

## 关联知识点

- `11_@RequestBody请求体`：请求体的反序列化
- `13_@RestController复合注解`：@ResponseBody 的组合
