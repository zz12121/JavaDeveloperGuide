# @ResponseBody响应体 - QA

## Q1：@ResponseBody 和 @RestController 哪个更好？

**A**：根据场景选择。

| 场景 | 推荐 |
|------|------|
| RESTful API（全部返回 JSON） | `@RestController` |
| 同时有页面跳转和接口 | `@Controller` + 方法级 `@ResponseBody` |

```java
// 场景1：纯 API
@RestController
@RequestMapping("/api/user")
public class UserApiController {
    // 全部返回 JSON，无需单独加 @ResponseBody
    @GetMapping("/{id}")
    public User getUser(@PathVariable Long id) { return userService.getById(id); }
}

// 场景2：混合项目
@Controller
@RequestMapping("/user")
public class UserController {
    @GetMapping("/list")
    public String listPage() {
        return "user/list";  // 返回视图名
    }
    
    @GetMapping("/{id}")
    @ResponseBody
    public User getUser(@PathVariable Long id) {
        return userService.getById(id);  // 返回 JSON
    }
}
```

---

## Q2：返回中文乱码怎么办？

**A**：三种解决方案。

```java
// 方案1：注解指定（局部）
@GetMapping(value = "/user/{id}", produces = "application/json;charset=UTF-8")

// 方案2：配置文件（全局）
spring:
  http:
    encoding:
      charset: UTF-8
      enabled: true
      force: true

// 方案3：配置类（全局）
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {
    @Override
    public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
        converters.add(0, new StringHttpMessageConverter(StandardCharsets.UTF_8));
    }
}
```

---

## Q3：返回 null 会怎样？

**A**：响应体为空。

| 情况 | 响应状态码 |
|------|-----------|
| 返回 null | 200 OK（空响应体） |
| 方法抛异常 | 由异常处理器处理 |
| @ResponseStatus 自定义 | 自定义状态码 |

```java
// 返回 null 的场景
@GetMapping("/{id}")
@ResponseBody
public User getUser(@PathVariable Long id) {
    if (!exists(id)) {
        return null;  // 返回空响应
    }
    return userService.getById(id);
}

// 建议：返回 404
@GetMapping("/{id}")
@ResponseBody
public ResponseEntity<User> getUser(@PathVariable Long id) {
    User user = userService.getById(id);
    if (user == null) {
        return ResponseEntity.notFound().build();
    }
    return ResponseEntity.ok(user);
}
```

---

## Q4：如何自定义响应格式？

**A**：通过 Result 统一响应结构。

```java
// 统一响应类
public class Result<T> {
    private int code;
    private String message;
    private T data;
    private long timestamp;
    
    public static <T> Result<T> success(T data) {
        Result<T> r = new Result<>();
        r.setCode(200);
        r.setMessage("success");
        r.setData(data);
        r.setTimestamp(System.currentTimeMillis());
        return r;
    }
    
    public static <T> Result<T> error(String message) {
        Result<T> r = new Result<>();
        r.setCode(500);
        r.setMessage(message);
        return r;
    }
}
```

---

## Q5：@ResponseBody 和 ResponseEntity 的区别？

**A**：

| 特性 | @ResponseBody | ResponseEntity |
|------|---------------|----------------|
| 用途 | 返回响应体 | 返回完整响应 |
| 状态码 | 默认 200 | 可自定义 |
| 响应头 | 默认 | 可自定义 |
| 灵活性 | 较低 | 高 |

```java
// @ResponseBody：简单场景
@GetMapping("/{id}")
@ResponseBody
public User getUser(@PathVariable Long id) {
    return userService.getById(id);
}

// ResponseEntity：复杂场景
@GetMapping("/{id}")
public ResponseEntity<User> getUser(@PathVariable Long id) {
    User user = userService.getById(id);
    if (user == null) {
        return ResponseEntity.notFound().build();
    }
    HttpHeaders headers = new HttpHeaders();
    headers.add("X-Custom-Header", "value");
    return ResponseEntity.ok().headers(headers).body(user);
}
```
