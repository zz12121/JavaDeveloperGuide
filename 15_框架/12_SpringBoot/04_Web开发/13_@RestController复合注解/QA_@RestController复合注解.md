# @RestController复合注解 - QA

## Q1：@RestController 和 @Controller + @ResponseBody 等价吗？

**A**：基本等价，但有细微区别。

```java
// 方式1：@RestController
@RestController
@RequestMapping("/api/user")
public class UserController {}

// 方式2：@Controller + @ResponseBody
@Controller
@ResponseBody
@RequestMapping("/api/user")
public class UserController {}
```

**区别**：
- `@RestController` 是组合注解，代码更简洁
- `@RestController` 的 value 属性会传递给 `@Controller`
- 功能上完全等价

---

## Q2：@RestController 能返回视图吗？

**A**：不能。

**解决方案**：

```java
// ❌ @RestController 不支持视图
@RestController
public class UserController {
    @GetMapping("/page")
    public String page() {
        return "user/list";  // 返回 "user/list" 字符串，不是视图
    }
}

// ✅ 改用 @Controller
@Controller
public class UserController {
    @GetMapping("/page")
    public String page() {
        return "user/list";  // 正确解析为视图
    }
}

// ✅ 或混合使用
@Controller
@RestController
public class MixedController {
    // @GetMapping → 视图
    // @GetMapping + @ResponseBody → JSON
}
```

---

## Q3：@RestController 需要配合 @RequestMapping 吗？

**A**：不需要。

```java
// 类级别 @RequestMapping（推荐）
@RestController
@RequestMapping("/api/user")
public class UserController {
    @GetMapping("/{id}")
    public User getUser(@PathVariable Long id) { return userService.getById(id); }
}

// 没有 @RequestMapping
@RestController
public class SimpleController {
    @GetMapping("/hello")
    public String hello() { return "Hello"; }  // /hello
}
```

---

## Q4：@RestController 的方法能接收 Model 参数吗？

**A**：可以，但意义不大。

```java
@RestController
public class UserController {
    // Model 参数存在，但不会被使用
    @GetMapping("/{id}")
    public User getUser(@PathVariable Long id, Model model) {
        // model 数据不会传递到视图
        return userService.getById(id);
    }
}
```

**建议**：RESTful API 中不使用 Model，改为返回数据对象。

---

## Q5：Spring Boot 推荐用 @RestController 吗？

**A**：是的，RESTful API 的首选。

**推荐**：
```java
@RestController
@RequestMapping("/api/users")
public class UserRestController {
    @GetMapping("/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.getById(id);
    }
    
    @PostMapping
    public User create(@RequestBody @Valid User user) {
        return userService.save(user);
    }
}
```

**不适合**：
- 需要返回 JSP/Thymeleaf 页面
- 需要 Model 数据传递到视图
- 需要重定向

**适合**：
- JSON API
- 前后端分离
- 微服务接口
