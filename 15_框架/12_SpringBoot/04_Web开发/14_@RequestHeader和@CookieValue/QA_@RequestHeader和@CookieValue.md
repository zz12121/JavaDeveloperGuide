# @RequestHeader和@CookieValue - QA

## Q1：@RequestHeader 获取不到值会怎样？

**A**：

| 情况 | 行为 |
|------|------|
| required=true（默认） | 400 Bad Request |
| required=false | null |
| 设置 defaultValue | 使用默认值 |

```java
// 默认：必须存在
@RequestHeader("X-Custom") String header;

// 可选：不存在返回 null
@RequestHeader(value = "X-Custom", required = false) String header;

// 默认值
@RequestHeader(value = "X-Custom", defaultValue = "default") String header;
```

---

## Q2：如何获取所有请求头？

**A**：

```java
// Map 方式
@GetMapping("/headers")
public Map<String, String> getHeaders(@RequestHeader Map<String, String> headers) {
    return headers;
}

// HttpHeaders 方式
@GetMapping("/headers")
public Map<String, List<String>> getHeaders(@RequestHeader HttpHeaders headers) {
    Map<String, List<String>> result = new HashMap<>();
    result.put("Accept", headers.getAccept());
    result.put("Accept-Language", headers.getAcceptLanguage());
    return result;
}
```

---

## Q3：@CookieValue 获取不到 Cookie 会怎样？

**A**：与 @RequestHeader 类似。

```java
// 默认：必须存在
@CookieValue("session_id") String sessionId;

// 可选
@CookieValue(value = "session_id", required = false) String sessionId;
```

---

## Q4：Cookie 值是对象而不是字符串怎么办？

**A**：使用 HttpServletRequest 手动解析。

```java
@GetMapping("/cookies")
public Map<String, Object> getCookies(HttpServletRequest request) {
    Map<String, Object> result = new HashMap<>();
    
    if (request.getCookies() != null) {
        for (Cookie cookie : request.getCookies()) {
            result.put(cookie.getName(), cookie.getValue());
        }
    }
    
    return result;
}
```

---

## Q5：@RequestHeader 和 @RequestParam 的区别？

**A**：

| 对比 | @RequestHeader | @RequestParam |
|------|----------------|---------------|
| 数据来源 | 请求头 | URL 参数/表单 |
| 示例 | `Accept: application/json` | `?name=value` |
| 使用场景 | 认证、日志、区分客户端 | 业务参数 |

```java
@GetMapping("/example")
public String example(
    @RequestHeader("Authorization") String auth,  // 请求头
    @RequestParam("name") String name             // 查询参数
) {
    return "Auth: " + auth + ", Name: " + name;
}
```
