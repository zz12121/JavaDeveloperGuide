# @RequestMapping请求映射 - QA

## Q1：@RequestMapping 和 @GetMapping 有什么区别？

**A**：

| 对比 | @RequestMapping | @GetMapping |
|------|-----------------|-------------|
| 适用范围 | 类和方法 | 仅方法 |
| HTTP 方法 | 默认所有方法 | 仅 GET |
| 使用场景 | 需要细粒度控制 | 普通查询接口 |
| 代码简洁 | 需要指定 method | 自动限定 GET |

```java
// @RequestMapping 需要指定 method
@RequestMapping(value = "/user", method = RequestMethod.GET)

// @GetMapping 简写，自动限定 GET
@GetMapping("/user")

// 其他快捷注解
@PostMapping("/user")       // POST
@PutMapping("/user/{id}")   // PUT
@DeleteMapping("/user/{id}") // DELETE
@PatchMapping("/user/{id}") // PATCH
```

---

## Q2：类级别的 @RequestMapping 有什么用？

**A**：为所有方法添加公共前缀。

```java
@Controller
@RequestMapping("/api/user")
public class UserController {
    
    @GetMapping("/list")     // → /api/user/list
    @GetMapping("/{id}")     // → /api/user/{id}
    @PostMapping("/save")    // → /api/user/save
}
```

**好处**：
- 减少重复路径书写
- 统一管理模块路径
- 便于版本管理 `/v1/xxx`、`/v2/xxx`

---

## Q3：如何处理多个路径映射到同一方法？

**A**：使用数组：

```java
@RequestMapping(value = {"/user", "/users", "/u"}, method = RequestMethod.GET)
@ResponseBody
public List<User> list() {
    return userService.list();
}
```

---

## Q4：路径匹配规则是什么？

**A**：

| 通配符 | 说明 | 示例 |
|--------|------|------|
| `?` | 任意单个字符 | `/user/?` → `/user/1` |
| `*` | 任意一路径段 | `/user/*` → `/user/1` |
| `**` | 任意路径 | `/user/**` → `/user/1/2/3` |
| `{var}` | 路径变量 | `/user/{id}` → `/user/1` |
| `{var:regex}` | 正则匹配 | `/user/{id:\\d+}` |

```java
// 正则路径变量
@GetMapping("/user/{id:\\d+}")
public User getUser(@PathVariable Long id) {}

// 多个路径变量
@GetMapping("/user/{userId}/order/{orderId}")
public Order getOrder(@PathVariable Long userId, @PathVariable Long orderId) {}
```

---

## Q5：params 和 headers 条件有什么用？

**A**：用于更细粒度的请求筛选。

```java
// 参数条件
@GetMapping(value = "/list", params = "type=1")
public List<User> listV1() {}  // ?type=1 才匹配

@GetMapping(value = "/list", params = "type=2")
public List<User> listV2() {}  // ?type=2 才匹配

// 请求头条件
@GetMapping(value = "/html", headers = "Accept=text/html")
public String htmlPage() {}

@GetMapping(value = "/json", headers = "Accept=application/json")
@ResponseBody
public List<User> jsonPage() {}
```

**场景**：
- 同一路径不同参数返回不同格式
- API 版本控制
- 区分移动端/PC端
