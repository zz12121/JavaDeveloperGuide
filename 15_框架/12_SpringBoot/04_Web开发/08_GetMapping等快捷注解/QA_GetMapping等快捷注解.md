# GetMapping等快捷注解 - QA

## Q1：@GetMapping 和 @RequestMapping(method = GET) 完全等价吗？

**A**：几乎等价，但不完全。

| 特性 | @RequestMapping | @GetMapping |
|------|-----------------|-------------|
| HTTP 方法限定 | ✅ | ✅ |
| 路径映射 | ✅ | ✅ |
| 继承其他属性 | ⚠️ 仅 method | ⚠️ 仅 method |
| 其他 HTTP 方法 | ❌ 不支持 | ❌ 不支持 |

**差异**：注解元数据略有不同，但运行时行为一致。

---

## Q2：RESTful 中各 HTTP 方法的语义是什么？

**A**：

| 方法 | 语义 | 幂等性 | 安全性 |
|------|------|--------|--------|
| GET | 查询资源 | ✅ 幂等 | ✅ 安全 |
| POST | 创建资源 | ❌ 非幂等 | ❌ 不安全 |
| PUT | 更新资源（完整） | ✅ 幂等 | ❌ 不安全 |
| DELETE | 删除资源 | ✅ 幂等 | ❌ 不安全 |
| PATCH | 部分更新资源 | ❌ 非幂等 | ❌ 不安全 |

**幂等**：多次执行结果相同
**安全**：不修改服务器资源

---

## Q3：@GetMapping 能接收请求体吗？

**A**：技术上可以，但**不应该**。

| 客户端 | 行为 |
|--------|------|
| 浏览器 | 通常不带 body |
| Postman/curl | 可以发送，但不推荐 |
| HttpClient | 会被拒绝或忽略 |

**规范**：
```java
// ❌ 不推荐：GET 请求带 body
@GetMapping("/search")
public List<User> search(@RequestBody SearchCondition condition) {}

// ✅ 推荐：使用查询参数
@GetMapping("/search")
public List<User> search(SearchCondition condition) {}

// ✅ 或者
@GetMapping("/search")
public List<User> search(
    @RequestParam String name,
    @RequestParam Integer age
) {}
```

---

## Q4：PUT 和 PATCH 的区别？

**A**：

| 方法 | 场景 | 示例 |
|------|------|------|
| PUT | 完整更新，传入所有字段 | `User{name="new", age=25}` |
| PATCH | 部分更新，只传变更字段 | `User{name="new"}`（age 不变） |

```java
// PUT：必须提供所有字段
@PutMapping("/{id}")
public User fullUpdate(@PathVariable Long id, @RequestBody User user) {
    user.setId(id);
    return userService.replace(user);  // 全量替换
}

// PATCH：只需提供变更字段
@PatchMapping("/{id}")
public User partialUpdate(@PathVariable Long id, @RequestBody Map<String, Object> updates) {
    return userService.patch(id, updates);  // 部分更新
}
```

---

## Q5：为什么 DELETE 返回 200 而不是 204？

**A**：取决于是否返回响应体。

| 返回 | HTTP 状态码 | 场景 |
|------|-------------|------|
| 返回删除的资源 | 200 OK | 需要返回删除的数据 |
| 无响应体 | 204 No Content | 仅确认删除 |
| 返回空 | 200 OK | 默认情况 |

```java
@DeleteMapping("/{id}")
@ResponseStatus(HttpStatus.NO_CONTENT)  // 明确返回 204
public void delete(@PathVariable Long id) {
    userService.deleteById(id);
}
```
