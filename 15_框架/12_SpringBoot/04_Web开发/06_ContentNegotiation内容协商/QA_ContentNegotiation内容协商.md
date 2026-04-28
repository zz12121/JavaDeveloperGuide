# ContentNegotiation内容协商 - QA

## Q1：如何让接口同时返回 JSON 和 XML？

**A**：两种方式：

```java
// 方式1：使用内容协商（自动）
@RestController
public class UserController {
    @GetMapping("/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.getById(id);
    }
}
// 请求: curl -H "Accept: application/json" → JSON
// 请求: curl -H "Accept: application/xml" → XML

// 方式2：使用 @RequestMapping 指定
@GetMapping(value = "/{id}", produces = {"application/json", "application/xml"})
public User getUser(@PathVariable Long id) {
    return userService.getById(id);
}
```

**注意**：XML 需要 JAXB 依赖：
```xml
<dependency>
    <groupId>com.fasterxml.jackson.dataformat</groupId>
    <artifactId>jackson-dataformat-xml</artifactId>
</dependency>
```

---

## Q2：Postman 请求 JSON 和浏览器访问行为不同？

**A**：是的，这是正常现象。

| 客户端 | Accept 头 | 结果 |
|--------|-----------|------|
| Postman | `application/json` | JSON |
| 浏览器 | `text/html,application/xhtml+xml` | HTML |
| curl | `*/*` | 默认 JSON |

**解决方案**：
```yaml
# 关闭浏览器友好的内容协商
spring:
  mvc:
    contentnegotiation:
      favor-parameter: false
      favor-path-extension: false
```

---

## Q3：如何通过参数指定返回格式？

**A**：启用参数协商：

```java
@Override
public void configureContentNegotiation(ContentNegotiationConfigurer configurer) {
    configurer
        .favorParameter(true)
        .parameterName("format");
}
```

```bash
# 使用参数
GET /api/user/1?format=json   → JSON
GET /api/user/1?format=xml    → XML
```

---

## Q4：如何让 JSON 优先于 XML？

**A**：配置顺序决定优先级：

```java
@Override
public void configureContentNegotiation(ContentNegotiationConfigurer configurer) {
    configurer
        .mediaTypes("json", MediaType.APPLICATION_JSON)
        .mediaTypes("xml", MediaType.APPLICATION_XML);
    // JSON 在前，优先匹配
}
```

---

## Q5：@Produces 和内容协商有什么关系？

**A**：`@Produces` 是内容协商的固定方式：

```java
@GetMapping(value = "/{id}", produces = MediaType.APPLICATION_JSON_VALUE)
public User getUser(@PathVariable Long id) {
    return userService.getById(id);
}
```

**对比**：
| 方式 | 说明 |
|------|------|
| `@Produces` | 固定返回格式，忽略 Accept 头 |
| 内容协商 | 根据 Accept 头动态选择格式 |
