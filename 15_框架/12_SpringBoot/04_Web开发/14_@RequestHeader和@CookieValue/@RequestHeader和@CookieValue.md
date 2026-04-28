# @RequestHeader和@CookieValue

## 先说结论

`@RequestHeader` 提取请求头数据，`@CookieValue` 提取 Cookie 数据，用于获取 HTTP 请求的元信息。

## 深度解析

### @RequestHeader 注解

```java
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface RequestHeader {
    @AliasFor("value")
    String name() default "";
    
    @AliasFor("name")
    String value() default "";
    
    boolean required() default true;
    
    String defaultValue() default ValueConstants.DEFAULT_NONE;
}
```

### @CookieValue 注解

```java
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface CookieValue {
    @AliasFor("value")
    String name() default "";
    
    @AliasFor("name")
    String value() default "";
    
    boolean required() default true;
    
    String defaultValue() default ValueConstants.DEFAULT_NONE;
}
```

### 常用请求头

| 请求头 | 说明 |
|--------|------|
| `Accept` | 客户端可接受的媒体类型 |
| `Content-Type` | 请求体的媒体类型 |
| `Authorization` | 认证信息 |
| `User-Agent` | 用户代理 |
| `X-Request-ID` | 请求唯一标识 |

## 代码示例

### 获取请求头

```java
@GetMapping("/header")
public String getHeader(
    @RequestHeader("Accept") String accept,
    @RequestHeader("User-Agent") String userAgent,
    @RequestHeader(value = "X-Custom-Header", required = false) String customHeader
) {
    return "Accept: " + accept + ", User-Agent: " + userAgent;
}
```

### 获取所有请求头

```java
@GetMapping("/headers")
public Map<String, String> getAllHeaders(@RequestHeader Map<String, String> headers) {
    return headers;
}
```

### 获取 Cookie

```java
@GetMapping("/cookie")
public String getCookie(@CookieValue("JSESSIONID") String sessionId) {
    return "Session ID: " + sessionId;
}

// 获取所有 Cookie
@GetMapping("/cookies")
public Map<String, String> getCookies(@CookieValue Map<String, String> cookies) {
    return cookies;
}
```

### 实际应用场景

```java
@PostMapping("/upload")
public Result upload(
    @RequestHeader("Content-Type") String contentType,
    @RequestParam("file") MultipartFile file
) {
    if (!contentType.startsWith("multipart/")) {
        return Result.error("需要使用 multipart/form-data");
    }
    return Result.success(fileService.upload(file));
}
```

## 关联知识点

- `07_@RequestMapping请求映射`：请求映射基础
- `10_@RequestParam请求参数`：请求参数提取
