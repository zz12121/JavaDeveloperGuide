# RedirectAttributes重定向数据 - QA

## Q1：addAttribute 和 addFlashAttribute 的区别？

**A**：

| 对比 | addAttribute | addFlashAttribute |
|------|--------------|-------------------|
| 数据存储 | URL 参数 | Session（FlashMap） |
| URL 显示 | ✅ 显示 | ❌ 不显示 |
| 数据保留 | 仅当前请求 | 重定向完成后 |
| 适用场景 | 需要在 URL 保留参数 | 显示一次性消息 |

```java
// addAttribute：参数显示在 URL
redirectAttributes.addAttribute("id", 123);
redirectAttributes.addAttribute("type", "success");
return "redirect:/user/123";
// URL: /user/123?id=123&type=success

// addFlashAttribute：参数隐藏在 Session
redirectAttributes.addFlashAttribute("message", "保存成功");
return "redirect:/user/list";
// URL: /user/list
// Session 中存储，重定向后取出显示
```

---

## Q2：Flash 属性在视图中如何获取？

**A**：通过 Model 或 ModelMap。

```java
@GetMapping("/list")
public String list(Model model) {
    // Flash 属性存储在 Model 中
    // ${message} 可直接访问
    return "user/list";
}
```

```html
<!-- Thymeleaf -->
<div th:if="${message}" th:text="${message}"></div>

<!-- JSP -->
<c:if test="${not empty message}">
    ${message}
</c:if>
```

---

## Q3：重定向时传递复杂对象怎么办？

**A**：

```java
// ❌ 不推荐：复杂对象不能作为 URL 参数
redirectAttributes.addAttribute("user", user);

// ✅ 推荐：使用 Flash 属性
redirectAttributes.addFlashAttribute("user", user);

// ✅ 或：序列化后传递
redirectAttributes.addAttribute("userJson", JSON.toJSONString(user));
return "redirect:/user/detail";
```

---

## Q4：Flash 属性会泄漏数据吗？

**A**：Flash 属性存储在 Session 中，重定向后自动清除。

**注意**：
- 避免存储敏感信息
- 大量 Flash 属性会增加 Session 负担
- 浏览器回退可能重新显示

---

## Q5：如何在 @RestController 中实现重定向？

**A**：使用 ResponseEntity。

```java
@RestController
@RequestMapping("/api/user")
public class UserRestController {
    
    @PostMapping
    public ResponseEntity<Void> create(@RequestBody User user) {
        User saved = userService.save(user);
        
        HttpHeaders headers = new HttpHeaders();
        headers.setLocation(URI.create("/api/user/" + saved.getId()));
        return new ResponseEntity<>(headers, HttpStatus.CREATED);
    }
}
```
