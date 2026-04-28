# @RequestBody请求体 - QA

## Q1：@RequestBody 和 @RequestParam 的区别？

**A**：

| 对比 | @RequestBody | @RequestParam |
|------|--------------|---------------|
| 数据来源 | HTTP 请求体 | URL 参数/表单 |
| Content-Type | application/json | application/x-www-form-urlencoded |
| 数据格式 | JSON/XML | key=value |
| 自动反序列化 | ✅ | ❌ |

```java
// @RequestBody 接收 JSON
@PostMapping("/save")
public Result save(@RequestBody User user) {
    // 请求体: {"name": "张三", "age": 25}
}

// @RequestParam 接收表单/查询参数
@PostMapping("/save")
public Result save(@RequestParam String name, @RequestParam Integer age) {
    // 参数: name=张三&age=25
}
```

---

## Q2：可以同时使用多个 @RequestBody 吗？

**A**：不可以。

**原因**：HTTP 请求只有一个请求体，无法拆分成多个对象。

**替代方案**：
```java
// 方案1：使用 Map
@PostMapping("/save")
public Result save(@RequestBody Map<String, Object> data) {
    User user = (User) data.get("user");
    Order order = (Order) data.get("order");
}

// 方案2：使用包装对象
@PostMapping("/save")
public Result save(@RequestBody SaveRequest request) {
    request.getUser();
    request.getOrder();
}
```

---

## Q3：@RequestBody 处理失败返回什么？

**A**：返回 400 Bad Request。

```json
{
    "timestamp": "2024-01-15T10:30:00.000+08:00",
    "status": 400,
    "error": "Bad Request",
    "message": "JSON parse error: ...",
    "path": "/api/user"
}
```

**自定义异常处理**：
```java
@ExceptionHandler(HttpMessageNotReadableException.class)
public Result handleNotReadable(HttpMessageNotReadableException e) {
    return Result.error("请求参数格式错误");
}
```

---

## Q4：如何处理未知属性？

**A**：

```java
// 配置 ObjectMapper（全局）
mapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);

// 或使用注解（局部）
public class User {
    @JsonIgnoreProperties(ignoreUnknown = true)
    private String extra;
}
```

---

## Q5：可以不用 @RequestBody 吗？

**A**：可以，但需要手动解析。

```java
// ❌ 不推荐：手动解析
@PostMapping("/save")
public Result save(HttpServletRequest request) {
    String body = request.getReader().lines().collect(Collectors.joining());
    User user = JSON.parseObject(body, User.class);
}

// ✅ 推荐：使用 @RequestBody
@PostMapping("/save")
public Result save(@RequestBody User user) {
    return Result.success(userService.save(user));
}
```
