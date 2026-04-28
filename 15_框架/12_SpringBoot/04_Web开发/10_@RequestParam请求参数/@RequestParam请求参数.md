# @RequestParam请求参数

## 先说结论

`@RequestParam` 用于获取 URL 查询参数（`?key=value`）或 POST 表单数据，弥补路径变量无法处理可选参数的不足。

## 深度解析

### 注解源码

```java
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface RequestParam {
    @AliasFor("value")
    String name() default "";
    
    @AliasFor("name")
    String value() default "";
    
    boolean required() default true;
    
    String defaultValue() default ValueConstants.DEFAULT_NONE;
}
```

### 基本使用

```java
@GetMapping("/user")
public List<User> search(
    @RequestParam String name,
    @RequestParam Integer age
) {
    return userService.search(name, age);
}

// GET /user?name=张三&age=25
```

### 核心属性

| 属性 | 类型 | 说明 |
|------|------|------|
| `name/value` | String | 参数名 |
| `required` | boolean | 是否必需，默认 true |
| `defaultValue` | String | 默认值 |

### 常用场景

```java
// 1. 简单参数
@GetMapping("/user")
public User getUser(@RequestParam Long id) {}

// 2. 指定参数名（与变量名不同时）
@GetMapping("/user")
public User getUser(@RequestParam("user_id") Long id) {}

// 3. 设置默认值
@GetMapping("/user")
public List<User> list(@RequestParam(defaultValue = "10") Integer size) {}

// 4. 可选参数
@GetMapping("/user")
public List<User> list(@RequestParam(required = false) String name) {
    if (name == null) {
        return userService.list();
    }
    return userService.search(name);
}

// 5. 多值参数
@GetMapping("/user")
public List<User> search(@RequestParam List<Long> ids) {
    return userService.getByIds(ids);
}
// GET /user?ids=1&ids=2&ids=3
```

## 易错点/踩坑

- ❌ 不传必需参数 → 400 Bad Request: Required parameter 'xxx' is not present
- ❌ 使用 `defaultValue` 但同时 `required=true` → 冲突，required 失效
- ❌ 使用对象接收查询参数 → 不会自动绑定，需要 `@ModelAttribute`
- ❌ 表单提交用 `@RequestParam` → 对于 multipart/form-data 可以，application/x-www-form-urlencoded 不推荐

## 代码示例

### 分页查询

```java
@GetMapping("/user/page")
public Page<User> page(
    @RequestParam(defaultValue = "1") Integer pageNum,
    @RequestParam(defaultValue = "10") Integer pageSize,
    @RequestParam(required = false) String keyword
) {
    return userService.page(pageNum, pageSize, keyword);
}

// GET /user/page?pageNum=2&pageSize=20&keyword=张
```

### 排序

```java
@GetMapping("/user")
public List<User> list(
    @RequestParam(defaultValue = "id") String sort,
    @RequestParam(defaultValue = "asc") String order
) {
    return userService.list(sort, order);
}

// GET /user?sort=createTime&order=desc
```

### 多值参数

```java
// 数组
@GetMapping("/user")
public List<User> getByIds(@RequestParam Long[] ids) {}

// 列表
@GetMapping("/user")
public List<User> getByIds(@RequestParam List<Long> ids) {}

// Map
@GetMapping("/search")
public List<User> search(@RequestParam Map<String, String> params) {
    String name = params.get("name");
    String email = params.get("email");
    return userService.search(name, email);
}

// GET /search?name=张三&email=zhang@example.com
```

## 图解/流程

```
请求: GET /user?name=张三&age=25&hobby=篮球&hobby=足球

@PathVariable 不可用 ❌（不适合可选参数）

@RequestParam 可用 ✅
├── @RequestParam String name   → "张三"
├── @RequestParam Integer age   → 25
├── @RequestParam List<String> hobby → ["篮球", "足球"]
└── @RequestParam Map<String, String> → 全部参数
```

## 关联知识点

- `09_@PathVariable路径变量`：路径变量与查询参数的区别
- `11_@RequestBody请求体`：处理 JSON 请求体
- `15_@ModelAttribute接收表单`：接收表单数据
