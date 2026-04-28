# @RestController复合注解

## 先说结论

`@RestController` = `@Controller` + `@ResponseBody`，是 Spring 4.0 引入的注解，专为 RESTful API 设计。**每个方法自动返回 JSON，无需显式添加 @ResponseBody**。

## 深度解析

### 注解源码

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Controller
@ResponseBody
public @interface RestController {
    @AliasFor(annotation = Controller.class)
    String value() default "";
}
```

### 与 @Controller 的对比

| 特性 | @Controller | @RestController |
|------|-------------|------------------|
| 视图解析 | ✅ 支持 | ❌ 不支持 |
| @ResponseBody | 需要显式添加 | 自动添加 |
| 返回值处理 | 视图名 | 响应体 |
| 使用场景 | 传统 Web | RESTful API |

### 工作原理

```
@RestController
        ↓
标注 @Controller → Spring 扫描并注册为 Bean
        ↓
标注 @ResponseBody → 每个方法返回值写入响应体
        ↓
无需视图解析器
```

## 易错点/踩坑

- ❌ @RestController + return "redirect:/xxx" → 不会重定向，直接返回字符串
- ❌ @RestController 返回视图名 → 需要改用 @Controller
- ❌ @RestController 使用 Model → 返回值会被忽略

## 代码示例

### 基本使用

```java
@RestController
@RequestMapping("/api/user")
public class UserController {
    
    @GetMapping("/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.getById(id);
        // 无需 @ResponseBody，自动返回 JSON
    }
}
```

### 需要重定向时

```java
@RestController
public class SomeController {
    
    // 使用 ResponseEntity 实现重定向
    @PostMapping("/submit")
    public ResponseEntity<Void> submit(@RequestBody Data data) {
        // 处理提交
        return ResponseEntity.status(HttpStatus.FOUND)
                .header("Location", "/result")
                .build();
    }
}
```

### 混合使用：同时支持页面和接口

```java
@Controller
public class MixedController {
    
    // 返回视图
    @GetMapping("/page")
    public String page(Model model) {
        model.addAttribute("users", userService.list());
        return "user/list";
    }
    
    // 返回 JSON
    @GetMapping("/api/users")
    @ResponseBody
    public List<User> apiList() {
        return userService.list();
    }
}
```

## 图解/流程

```
@RestController
        ↓
┌─────────────────────────────────────┐
│ @Controller                         │
│   → Bean 注册                       │
│   → 组件扫描发现                     │
├─────────────────────────────────────┤
│ @ResponseBody                       │
│   → 方法返回值 → 响应体               │
│   → 自动 JSON 序列化                │
└─────────────────────────────────────┘
        ↓
直接返回 JSON，不走视图解析
```

## 关联知识点

- `12_@ResponseBody响应体`：@ResponseBody 详解
- `07_@RequestMapping请求映射`：请求映射基础
