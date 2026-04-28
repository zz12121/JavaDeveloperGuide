# @PathVariable路径变量

## 先说结论

`@PathVariable` 用于从 URL 路径中提取变量，配合 `@RequestMapping` 的 `{变量名}` 一起使用，是 RESTful API 的核心。

## 深度解析

### 注解源码

```java
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface PathVariable {
    @AliasFor("value")
    String name() default "";
    
    @AliasFor("name")
    String value() default "";
    
    boolean required() default true;  // Spring 4.3+
}
```

### 基本使用

```java
@GetMapping("/user/{id}")
public User getUser(@PathVariable Long id) {
    return userService.getById(id);
}
```

### 路径变量与参数名匹配

```java
// 方式1：变量名与方法参数名一致（推荐）
@GetMapping("/user/{id}")
public User getUser(@PathVariable Long id) {}

// 方式2：变量名与方法参数名不一致
@GetMapping("/user/{userId}")
public User getUser(@PathVariable("userId") Long id) {}

// 方式3：省略 value，通过反射获取参数名（需要编译参数 -parameters）
@GetMapping("/user/{id}")
public User getUser(@PathVariable Long id) {}
```

### 多个路径变量

```java
@GetMapping("/user/{userId}/order/{orderId}")
public Order getOrder(
    @PathVariable Long userId,
    @PathVariable Long orderId
) {
    return orderService.getByUserAndId(userId, orderId);
}
```

### 可选路径变量（SpringBoot 2.x）

```java
// 方式1：required = false
@GetMapping({"/user/{id}", "/user"})
public User getUser(@PathVariable(required = false) Long id) {
    if (id == null) {
        return userService.getDefault();
    }
    return userService.getById(id);
}

// 方式2：使用 Optional
@GetMapping("/user/{id}")
public User getUser(@PathVariable Optional<Long> id) {
    return id.map(userService::getById).orElse(userService.getDefault());
}
```

## 易错点/踩坑

- ❌ 路径变量名与 @PathVariable 值不匹配 → 400 Bad Request
- ❌ 使用了中文路径变量 → URL 编码问题
- ❌ 路径变量为空 → `required=true` 时报错
- ❌ 路径变量类型转换失败 → 400 Bad Request

## 代码示例

### 正则约束

```java
// 只匹配数字 ID
@GetMapping("/user/{id:[0-9]+}")
public User getUser(@PathVariable Long id) {
    return userService.getById(id);
}

// 匹配字母用户名
@GetMapping("/user/{username:[a-zA-Z]+}")
public User getUser(@PathVariable String username) {
    return userService.getByUsername(username);
}
```

### 路径变量与请求参数共存

```java
@GetMapping("/user/{id}/list")
public List<Order> getOrders(
    @PathVariable Long id,
    @RequestParam(defaultValue = "10") Integer size
) {
    return orderService.getByUserId(id, size);
}

// GET /user/123/list?size=20
```

### 完整 RESTful 示例

```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    @GetMapping("/{id:\\d+}")
    public User getUser(@PathVariable Long id) {
        return userService.getById(id);
    }
    
    @GetMapping("/{id:\\d+}/posts/{postId:\\d+}")
    public Post getPost(
        @PathVariable Long id,
        @PathVariable Long postId
    ) {
        return postService.getByUserAndId(id, postId);
    }
    
    @GetMapping("/{userId:[a-z]{4}}/info")
    public UserInfo getUserInfo(@PathVariable String userId) {
        return userService.getInfo(userId);
    }
}
```

## 图解/流程

```
URL: GET /api/user/123/order/456
        ↓
@PathVariable("userId") → 123
@PathVariable("orderId") → 456
        ↓
执行方法：getOrder(Long userId, Long orderId)
        ↓
返回结果
```

## 关联知识点

- `07_@RequestMapping请求映射`：路径映射基础
- `10_@RequestParam请求参数`：查询参数与路径变量的区别
