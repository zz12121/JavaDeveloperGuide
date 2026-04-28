# @ModelAttribute接收表单 - QA

## Q1：@ModelAttribute 和 @RequestBody 的区别？

**A**：

| 对比 | @ModelAttribute | @RequestBody |
|------|-----------------|--------------|
| 数据格式 | 表单 (x-www-form-urlencoded) | JSON |
| 绑定方式 | 属性逐一绑定 | JSON 反序列化 |
| Content-Type | application/x-www-form-urlencoded | application/json |
| 适用场景 | 表单提交 | RESTful API |

```java
// 表单提交
@PostMapping("/save")
public String save(@ModelAttribute User user) {
    // name=张三&age=25 → user.setName("张三"), user.setAge(25)
}

// JSON 提交
@PostMapping("/save")
public Result save(@RequestBody User user) {
    // {"name": "张三", "age": 25} → user
}
```

---

## Q2：表单字段与对象属性名不一致怎么办？

**A**：使用 `@RequestParam` 逐一接收，或使用 `name` 属性映射。

```java
// 方式1：使用 name 属性
@PostMapping("/save")
public String save(
    @ModelAttribute User user,
    @RequestParam("user_name") String userName  // 表单字段名
) {
    user.setName(userName);
    return "success";
}

// 方式2：对象中添加 setter 别名
public class User {
    private String name;
    
    @ModelAttribute("user_name")
    public void setUserName(String name) {
        this.name = name;
    }
}
```

---

## Q3：@ModelAttribute 可以用在类级别吗？

**A**：可以，类级别的 @ModelAttribute 会为所有方法共享数据。

```java
@Controller
@ModelAttribute("currentUser")
public User getCurrentUser() {
    return getUserFromSession();
}

@RestController
@RequestMapping("/api")
public class ApiController {
    // 所有方法都可以访问 currentUser
    @GetMapping("/info")
    public UserInfo info(@ModelAttribute("currentUser") User user) {
        return userService.getInfo(user.getId());
    }
}
```

---

## Q4：@ModelAttribute 和 Model 的区别？

**A**：

| 对比 | @ModelAttribute | Model |
|------|-----------------|-------|
| 用途 | 绑定请求参数到对象 | 传递数据到视图 |
| 方向 | 请求 → 对象 | Controller → 视图 |
| 作用域 | 方法参数 | 方法参数 |

```java
@PostMapping("/save")
public String save(
    @ModelAttribute User user,  // 绑定表单到对象
    Model model                   // 传递数据到视图
) {
    userService.save(user);
    model.addAttribute("message", "保存成功");
    return "success";
}
```

---

## Q5：表单提交中文乱码怎么办？

**A**：检查以下配置。

```yaml
# application.yml
spring:
  http:
    encoding:
      charset: UTF-8
      enabled: true
      force: true
```

**Filter 方式**：
```java
@Configuration
public class EncodingConfig {
    @Bean
    public FilterRegistrationBean<CharacterEncodingFilter> encodingFilter() {
        CharacterEncodingFilter filter = new CharacterEncodingFilter();
        filter.setEncoding("UTF-8");
        filter.setForceEncoding(true);
        FilterRegistrationBean<CharacterEncodingFilter> bean = new FilterRegistrationBean<>();
        bean.setFilter(filter);
        return bean;
    }
}
```
