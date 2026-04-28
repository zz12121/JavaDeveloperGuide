# @RequestParam请求参数 - QA

## Q1：什么情况下 @RequestParam 可以省略？

**A**：满足以下条件时可以省略注解：

1. **简单类型参数**
2. **参数名与变量名一致**
3. **必需参数**

```java
// 省略前
@GetMapping("/user")
public User getUser(@RequestParam Long id) {}

// 省略后（推荐在简单场景使用）
@GetMapping("/user")
public User getUser(Long id) {}

// 注意：省略后要求参数名与变量名一致
// - 编译时加 -parameters
// - 或使用 -debug（获取参数名）
```

**建议**：显式使用 `@RequestParam`，代码更清晰。

---

## Q2：@RequestParam 和 @RequestParam(value, required) 哪个更好？

**A**：建议显式声明。

```java
// ✅ 推荐：明确声明意图
@RequestParam(name = "id", required = false)
private Long id;

// ⚠️ 可接受：使用别名（name=value）
@RequestParam(value = "id", defaultValue = "0")
private Long id;
```

---

## Q3：defaultValue 和 required=false 的区别？

**A**：

| 特性 | defaultValue | required=false |
|------|--------------|----------------|
| 参数缺失时 | 使用默认值 | 参数为 null |
| 类型 | 必须是常量字符串 | 可以是 null |
| 用途 | 有明确默认值 | 允许不传 |

```java
// 场景1：分页有默认值
@RequestParam(defaultValue = "10")
private Integer size;  // 不传 → 10

// 场景2：搜索关键字可选
@RequestParam(required = false)
private String keyword;  // 不传 → null
```

---

## Q4：POST 表单可以用 @RequestParam 吗？

**A**：可以，但建议区分场景。

| Content-Type | 接收方式 |
|--------------|----------|
| `application/x-www-form-urlencoded` | `@RequestParam` 或 `@ModelAttribute` |
| `multipart/form-data` | `@RequestParam` |
| `application/json` | `@RequestBody` |

```java
// 表单提交
@PostMapping("/save")
public Result save(
    @RequestParam String username,
    @RequestParam String password
) {}

// 文件上传
@PostMapping("/upload")
public Result upload(@RequestParam MultipartFile file) {}
```

---

## Q5：多个同名参数怎么处理？

**A**：

```java
// 方式1：数组
@GetMapping("/user")
public List<User> getByIds(@RequestParam Long[] ids) {
    // ids = [1, 2, 3]
}

// 方式2：List
@GetMapping("/user")
public List<User> getByIds(@RequestParam List<Long> ids) {
    // ids = [1, 2, 3]
}

// 方式3：Map
@GetMapping("/user")
public List<User> search(@RequestParam Map<String, String> params) {
    // params = {"ids": "1,2,3"} 或多个同名列
}
```

**前端传参**：
```javascript
// 方式1：多个同名参数
axios.get('/user', { params: { ids: [1, 2, 3] } })

// 方式2：逗号分隔
axios.get('/user', { params: { ids: '1,2,3' } })
```
