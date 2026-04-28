# RedirectAttributes重定向数据

## 先说结论

`RedirectAttributes` 是 Spring MVC 提供的重定向数据传递工具，支持 **URL 参数拼接**和** Flash 属性（临时存储）**两种方式。

## 深度解析

### RedirectAttributes 接口

```java
public interface RedirectAttributes {
    RedirectAttributes addAttribute(String key, Object value);
    RedirectAttributes addAttribute(Object value);
    RedirectAttributes addAllAttributes(Collection<?> values);
    RedirectAttributes addFlashAttribute(String key, Object value);
    RedirectAttributes addFlashAttribute(Object value);
    Map<String, ?> getFlashAttributes();
}
```

### 两种传值方式

| 方式 | 说明 | URL 显示 |
|------|------|----------|
| addAttribute | 拼接到 URL 参数 | ✅ 显示 |
| addFlashAttribute | 存储在 Session， redirect 后清除 | ❌ 不显示 |

### 基本使用

```java
@Controller
public class UserController {
    
    @PostMapping("/save")
    public String save(User user, RedirectAttributes redirectAttributes) {
        userService.save(user);
        
        // 方式1：URL 参数
        redirectAttributes.addAttribute("id", user.getId());
        redirectAttributes.addAttribute("success", true);
        return "redirect:/user/{id}";
        // 跳转后: /user/123?id=123&success=true
        
        // 方式2：Flash 属性（推荐，URL 更干净）
        redirectAttributes.addFlashAttribute("success", true);
        return "redirect:/user/" + user.getId();
    }
    
    @GetMapping("/{id}")
    public String detail(@PathVariable Long id, Model model) {
        model.addAttribute("user", userService.getById(id));
        return "user/detail";
    }
}
```

### Flash 属性场景

```java
// 保存成功后显示消息
@PostMapping("/save")
public String save(User user, RedirectAttributes redirectAttributes) {
    userService.save(user);
    
    // Flash 属性：重定向后通过 Model 获取
    redirectAttributes.addFlashAttribute("message", "保存成功！");
    return "redirect:/user/list";
}

@GetMapping("/list")
public String list(Model model) {
    model.addAttribute("users", userService.list());
    return "user/list";
}
```

```html
<!-- Thymeleaf 视图 -->
<div th:if="${message}" th:text="${message}" class="alert alert-success">
    <!-- 重定向后显示 "保存成功！" -->
</div>
```

## 易错点/踩坑

- ❌ @RestController 使用 RedirectAttributes → 不支持，@RestController 不走视图
- ❌ Flash 属性用于非重定向 → 不生效，Flash 属性只在重定向后有效
- ❌ 大量 Flash 属性 → 增加 Session 负担

## 代码示例

### RESTful 重定向

```java
@PostMapping
public ResponseEntity<Void> create(@RequestBody User user) {
    User saved = userService.save(user);
    HttpHeaders headers = new HttpHeaders();
    headers.add("Location", "/api/user/" + saved.getId());
    return new ResponseEntity<>(headers, HttpStatus.CREATED);
}
```

## 关联知识点

- `17_处理模型数据`：Model 数据处理
- `18_@SessionAttributes和@SessionAttribute`：会话数据
