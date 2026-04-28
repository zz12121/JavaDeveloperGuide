# WelcomePage欢迎页

## 先说结论

Spring Boot 支持欢迎页（index.html），访问根路径 `/` 时自动跳转到欢迎页。

## 深度解析

### 欢迎页位置

```
优先级顺序：
1. src/main/resources/static/index.html
2. src/main/resources/public/index.html
3. src/main/resources/templates/index.html（需模板引擎）
```

### 自定义欢迎页

```yaml
spring:
  mvc:
    welcome-page:
      enabled: true
```

### 控制器方式

```java
@Controller
public class HomeController {
    @GetMapping("/")
    public String home() {
        return "forward:/index.html";
    }
}
```

## 关联知识点

- `25_静态资源访问规则`：静态资源基础
