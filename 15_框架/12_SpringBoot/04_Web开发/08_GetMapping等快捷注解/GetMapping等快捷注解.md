# GetMapping等快捷注解

## 先说结论

Spring 4.3+ 提供了 `@GetMapping`、`@PostMapping`、`@PutMapping`、`@DeleteMapping`、`@PatchMapping` 五个快捷注解，**简化 @RequestMapping 的 method 属性**。

## 深度解析

### 注解继承关系

```java
// @GetMapping 源码
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@RequestMapping(method = RequestMethod.GET)  // 继承 GET 方法
public @interface GetMapping {
    // value, path, params, headers, consumes, produces 继承自 @RequestMapping
}
```

### 五个快捷注解

| 注解 | HTTP 方法 | 等价写法 |
|------|-----------|----------|
| `@GetMapping` | GET | `@RequestMapping(method = RequestMethod.GET)` |
| `@PostMapping` | POST | `@RequestMapping(method = RequestMethod.POST)` |
| `@PutMapping` | PUT | `@RequestMapping(method = RequestMethod.PUT)` |
| `@DeleteMapping` | DELETE | `@RequestMapping(method = RequestMethod.DELETE)` |
| `@PatchMapping` | PATCH | `@RequestMapping(method = RequestMethod.PATCH)` |

### 使用示例

```java
@RestController
@RequestMapping("/api/user")
public class UserController {
    
    @GetMapping       // 查询列表
    public List<User> list() { return Collections.emptyList(); }
    
    @GetMapping("/{id}")  // 查询单个
    public User get(@PathVariable Long id) { return new User(); }
    
    @PostMapping      // 新增
    public Result save(@RequestBody User user) { return Result.success(); }
    
    @PutMapping("/{id}")  // 修改
    public Result update(@PathVariable Long id, @RequestBody User user) { return Result.success(); }
    
    @DeleteMapping("/{id}")  // 删除
    public Result delete(@PathVariable Long id) { return Result.success(); }
}
```

## 易错点/踩坑

- ❌ 使用 `@GetMapping` 处理 POST 请求 → 405 Method Not Allowed
- ❌ `@GetMapping` 用于带请求体的操作 → GET 请求不应有 body，服务器可能忽略
- ❌ DELETE 请求使用 `@GetMapping` → 405 错误

## 代码示例

### 组合路径

```java
@GetMapping({"/list", "/all", "/"})
public List<User> list() {
    return userService.list();
}
```

### 带参数约束

```java
@PostMapping(consumes = "application/json", produces = "application/json")
public Result save(@RequestBody User user) {
    return Result.success();
}
```

### RESTful 最佳实践

```java
@RestController
@RequestMapping("/api/users")
public class UserRestController {
    
    // GET /api/users
    @GetMapping
    public List<User> getAll() { return userService.list(); }
    
    // GET /api/users/1
    @GetMapping("/{id}")
    public User getById(@PathVariable Long id) { return userService.getById(id); }
    
    // POST /api/users
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public User create(@RequestBody @Valid User user) { return userService.save(user); }
    
    // PUT /api/users/1
    @PutMapping("/{id}")
    public User modify(@PathVariable Long id, @RequestBody User user) {
        user.setId(id);
        return userService.update(user);
    }
    
    // DELETE /api/users/1
    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void remove(@PathVariable Long id) { userService.deleteById(id); }
}
```

## 图解/流程

```
HTTP 请求
    ↓
匹配 @GetMapping / @PostMapping / ...
    ↓
@RestController → 自动添加 @ResponseBody
    ↓
返回 JSON（无需手动加 @ResponseBody）
```

## 关联知识点

- `07_@RequestMapping请求映射`：父注解
- `09_@PathVariable路径变量`：路径变量提取
- `10_@RequestParam请求参数`：查询参数提取
