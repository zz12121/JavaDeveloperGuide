# HttpMessageConverters自动配置 - QA

## Q1：@RequestBody 和 @ResponseBody 的区别？

**A**：

| 注解 | 作用 | 触发 |
|------|------|------|
| `@RequestBody` | 将请求体转换为 Java 对象 | HttpMessageConverter **read** |
| `@ResponseBody` | 将 Java 对象转换为响应体 | HttpMessageConverter **write** |

```java
// @RequestBody: 请求体 → Java对象
@PostMapping("/save")
public Result save(@RequestBody User user) {
    // user 已自动从 JSON 转换为对象
    return Result.success();
}

// @ResponseBody: Java对象 → 响应体
@GetMapping("/get")
@ResponseBody
public User get() {
    return new User(); // 自动转换为 JSON
}
```

---

## Q2：如何指定使用哪个转换器？

**A**：

1. **根据 Content-Type 自动选择**：
   - `application/json` → Jackson
   - `application/xml` → JAXB
   - `text/plain` → String

2. **手动指定**：
```java
@PostMapping(value = "/save", consumes = "application/xml")
@ResponseBody
public Result save(@RequestBody User user) {
    return Result.success();
}
```

---

## Q3：Jackson 和 FastJson 哪个更快？

**A**：

| 指标 | Jackson | FastJson |
|------|---------|----------|
| 速度 | 较快 | 更快 |
| 功能 | 完善 | 完善 |
| 生态 | Spring 默认 | 阿里系常用 |
| 序列化注解 | `@JsonIgnore` | `@JSONField(serialize = false)` |

**建议**：
- 一般项目：使用默认 Jackson
- 高并发场景：考虑 FastJson
- 混用需谨慎，容易冲突

---

## Q4：如何配置全局日期格式？

**A**：三种方式：

```yaml
# 方式1：配置文件（Spring 2.0+）
spring:
  jackson:
    date-format: yyyy-MM-dd HH:mm:ss
    time-zone: GMT+8

# 方式2：配置类
@Configuration
public class JacksonConfig {
    @Bean
    public ObjectMapper objectMapper() {
        ObjectMapper mapper = new ObjectMapper();
        mapper.setDateFormat(new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));
        return mapper;
    }
}

# 方式3：注解（局部）
public class User {
    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss", timezone = "GMT+8")
    private Date createTime;
}
```

---

## Q5：转换失败返回什么？

**A**：默认返回 `400 Bad Request`。

**示例**：
```json
{
    "timestamp": "2024-01-15T10:30:00.000+08:00",
    "status": 400,
    "error": "Bad Request",
    "message": "JSON parse error: ...",
    "path": "/api/user"
}
```

**自定义处理**：
```java
@ControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(HttpMessageNotReadableException.class)
    public Result handleMessageNotReadable(HttpMessageNotReadableException e) {
        return Result.error("请求参数格式错误");
    }
}
```
