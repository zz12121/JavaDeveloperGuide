# @PathVariable路径变量 - QA

## Q1：@PathVariable 和 @RequestParam 的区别？

**A**：

| 特征 | @PathVariable | @RequestParam |
|------|---------------|---------------|
| 来源 | URL 路径 | URL 查询参数 |
| 示例 | `/user/{id}` | `/user?id=123` |
| RESTful 风格 | 资源标识 | 查询条件 |
| 可选性 | 需要配合正则 | 可设置默认值 |
| 顺序 | 必需参数 | 可选参数 |

```java
// @PathVariable：RESTful 风格
@GetMapping("/user/{id}")
public User getUser(@PathVariable Long id) {}

// @RequestParam：查询参数风格
@GetMapping("/user")
public User getUser(@RequestParam Long id) {}

// 组合使用
@GetMapping("/user/{id}/search")
public List<User> search(
    @PathVariable Long id,              // 路径变量
    @RequestParam String keyword,       // 查询参数
    @RequestParam(defaultValue = "10") int size
) {}
```

---

## Q2：@PathVariable 参数可以为空吗？

**A**：可以，使用 `required = false`。

```java
@GetMapping({"/user/{id}", "/user"})
public User getUser(@PathVariable(required = false) Long id) {
    if (id == null) {
        return userService.getDefault();
    }
    return userService.getById(id);
}
```

**注意**：路径需要使用 `{}` 占位，否则无法匹配。

---

## Q3：路径变量类型转换失败会怎样？

**A**：返回 400 Bad Request。

```java
@GetMapping("/user/{id}")
public User getUser(@PathVariable Long id) {}

// 请求：GET /user/abc
// 响应：400 Bad Request
// 原因："abc" 无法转换为 Long
```

**解决**：使用正则限制格式
```java
@GetMapping("/user/{id:[0-9]+}")
public User getUser(@PathVariable Long id) {}
```

---

## Q4：如何使用 Map 获取所有路径变量？

**A**：

```java
@GetMapping("/user/{id}/order/{orderId}")
public String getOrder(@PathVariable Map<String, String> pathVars) {
    String userId = pathVars.get("id");
    String orderId = pathVars.get("orderId");
    return "User: " + userId + ", Order: " + orderId;
}
```

---

## Q5：@PathVariable 支持哪些数据类型？

**A**：

| 类型 | 示例 | 说明 |
|------|------|------|
| String | `{id}` | 默认 |
| Long | `{id}` | 自动转换 |
| Integer | `{id}` | 自动转换 |
| Boolean | `{active}` | true/false |
| Double | `{price}` | 小数 |
| Date | `{date}` | 需要配置转换器 |

**自定义类型**：
```java
@GetMapping("/user/{user}")
public User getUser(@PathVariable User user) {
    // 需要 @PathVariable 和自定义 Converter
}
```

```java
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {
    @Override
    public void addFormatters(FormatterRegistry registry) {
        registry.addConverter(new Converter<String, User>() {
            @Override
            public User convert(String source) {
                User user = new User();
                user.setId(Long.parseLong(source));
                return user;
            }
        });
    }
}
```
