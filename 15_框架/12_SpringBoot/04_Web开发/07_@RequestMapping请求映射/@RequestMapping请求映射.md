# @RequestMapping请求映射

## 先说结论

`@RequestMapping` 是 Spring MVC 最基础的请求映射注解，用于将 URL 路径映射到处理方法。SpringBoot 自动配置了 `RequestMappingHandlerMapping` 来处理此注解。

## 深度解析

### 核心概念

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface RequestMapping {
    String name() default "";
    
    @AliasFor("path")
    String[] value() default {};
    
    @AliasFor("value")
    String[] path() default {};
    
    RequestMethod[] method() default {};
    
    String[] params() default {};
    
    String[] headers() default {};
    
    String[] consumes() default {};
    
    String[] produces() default {};
}
```

### 属性详解

| 属性 | 类型 | 说明 |
|------|------|------|
| `value/path` | String[] | URL 路径 |
| `method` | RequestMethod[] | 支持的 HTTP 方法 |
| `params` | String[] | 请求参数条件 |
| `headers` | String[] | 请求头条件 |
| `consumes` | String[] | 请求 Content-Type |
| `produces` | String[] | 响应 Content-Type |

### 使用示例

```java
@Controller
@RequestMapping("/api/user")
public class UserController {
    
    // 完整配置
    @RequestMapping(
        value = "/{id}",
        method = RequestMethod.GET,
        params = "version=1",
        headers = "X-Custom-Header=abc",
        consumes = "application/json",
        produces = "application/json"
    )
    @ResponseBody
    public User getUser(@PathVariable Long id) {
        return new User(id, "张三");
    }
    
    // 简化写法
    @GetMapping("/list")
    @ResponseBody
    public List<User> list() {
        return Collections.emptyList();
    }
}
```

## 易错点/踩坑

- ❌ 类级别 `@RequestMapping` 路径没有以 `/` 开头 → 相对路径，可能匹配错误
- ❌ 方法级别路径重复使用 `/` → `/api/user//list`
- ❌ `method` 不指定时，默认匹配所有 HTTP 方法
- ❌ `params` 条件不满足时，返回 415 或 400

## 代码示例

### 路径通配符

```java
@GetMapping("/user/*")      // 精确一个路径段
@GetMapping("/user/**")     // 任意路径段
@GetMapping("/user/??" )    // 任意两个字符
```

### 请求方法限定

```java
@RequestMapping(value = "/user", method = RequestMethod.GET)
@RequestMapping(value = "/user", method = RequestMethod.POST)
@RequestMapping(value = "/user", method = {RequestMethod.GET, RequestMethod.POST})
```

### 请求参数条件

```java
// 必须包含 version 参数
@RequestMapping(value = "/list", params = "version")

// 必须等于指定值
@RequestMapping(value = "/list", params = "version=1")

// 必须不包含
@RequestMapping(value = "/list", params = "!debug")
```

### 请求头条件

```java
// 必须包含指定请求头
@RequestMapping(value = "/html", headers = "Content-Type=text/html")

// 不能包含
@RequestMapping(value = "/json", headers = "!X-Custom-Header")
```

## 图解/流程

```
请求 URL: GET /api/user/1
        ↓
RequestMappingHandlerMapping 匹配
        ↓
1. 类 @RequestMapping("/api/user") + 方法 @GetMapping("/{id}")
   → 合并路径: /api/user/{id}

2. @RequestMapping(method = GET)
   → 方法匹配

3. @RequestMapping(params = "version=1")
   → 参数条件检查

4. @RequestMapping(headers = "X-Token")
   → 请求头检查

5. 全部通过 → 执行处理方法
   任一不满足 → 404 或 415
```

## 关联知识点

- `08_GetMapping等快捷注解`：@GetMapping 是 @RequestMapping 的快捷方式
- `09_@PathVariable路径变量`：路径中的变量提取
