# WelcomePage欢迎页 - QA

## Q1：欢迎页和 @RequestMapping("/") 冲突？

**A**：可能冲突。

**解决**：
```java
@Controller
public class HomeController {
    @RequestMapping("/")
    public String home() {
        return "redirect:/home";
    }
}
```

---

## Q2：欢迎页支持模板引擎吗？

**A**：支持。

```yaml
spring:
  thymeleaf:
    prefix: classpath:/templates/
    suffix: .html
```

在 `templates/index.html` 中使用 Thymeleaf 语法。

---

## Q3：如何禁用欢迎页？

**A**：

```yaml
spring:
  mvc:
    welcome-page:
      enabled: false
```
