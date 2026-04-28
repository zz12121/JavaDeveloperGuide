# @ModelAttribute接收表单

## 先说结论

`@ModelAttribute` 用于将请求参数绑定到模型对象，或者将模型数据传递到视图。**适合处理表单提交和复杂对象绑定**。

## 深度解析

### 用法分类

| 用法 | 位置 | 说明 |
|------|------|------|
| 方法参数 | Controller 方法参数 | 绑定请求参数到对象 |
| 方法级 | Controller 方法上 | 在每个方法前执行 |
| 类级 | Controller 类上 | 在所有方法前执行 |

### 基本使用

```java
// 绑定表单参数到对象
@PostMapping("/save")
public String save(@ModelAttribute("user") User user) {
    userService.save(user);
    return "redirect:/user/list";
}

// 简写：不指定 name
@PostMapping("/save")
public String save(User user) {
    // 自动以类名小写作为 name：user
    return "redirect:/user/list";
}
```

### @ModelAttribute 与表单绑定

```java
public class User {
    private Long id;
    private String name;
    private Integer age;
    private String email;
    // getters/setters
}

@PostMapping("/save")
public String save(@ModelAttribute User user) {
    // 表单字段自动绑定
    // name="name" → user.setName()
    // name="age" → user.setAge()
    userService.save(user);
    return "success";
}
```

### 方法级 @ModelAttribute

```java
@Controller
@RequestMapping("/user")
public class UserController {
    
    // 每个方法前都执行
    @ModelAttribute
    public void populateModel(Model model) {
        model.addAttribute("categories", categoryService.list());
    }
    
    @GetMapping("/form")
    public String form(Model model) {
        // categories 已存在
        return "user/form";
    }
    
    @PostMapping("/save")
    public String save(@ModelAttribute User user) {
        // categories 仍然存在
        return "redirect:/user/list";
    }
}
```

## 易错点/踩坑

- ❌ 表单字段名与对象属性名不匹配 → 参数绑定失败（使用 @ModelAttribute）
- ❌ GET 请求使用 @ModelAttribute → 不适用，GET 无请求体
- ❌ 方法级 @ModelAttribute 耗时操作 → 每个请求都执行，影响性能

## 代码示例

### 表单编辑场景

```java
@GetMapping("/edit/{id}")
public String edit(@PathVariable Long id, Model model) {
    model.addAttribute("user", userService.getById(id));
    return "user/edit";
}

@PostMapping("/update")
public String update(@ModelAttribute User user, Model model) {
    userService.update(user);
    return "redirect:/user/list";
}
```

### 数据校验

```java
@PostMapping("/save")
public String save(@ModelAttribute @Valid User user, BindingResult result) {
    if (result.hasErrors()) {
        return "user/form";
    }
    userService.save(user);
    return "redirect:/user/list";
}
```

## 关联知识点

- `10_@RequestParam请求参数`：简单参数接收
- `11_@RequestBody请求体`：JSON 请求体接收
- `17_处理模型数据`：Model 数据传递
