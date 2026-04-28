# @RequestBody请求体

## 先说结论

`@RequestBody` 将 HTTP 请求体中的数据（通常是 JSON）自动反序列化为 Java 对象，是 RESTful API 接收数据的标准方式。

## 深度解析

### 注解源码

```java
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface RequestBody {
    boolean required() default true;
}
```

### 工作原理

```
HTTP POST 请求
Content-Type: application/json

{"name": "张三", "age": 25}
        ↓
HttpMessageConverter（Jackson）
        ↓
反序列化为 Java 对象
        ↓
User{name="张三", age=25}
```

### 基本使用

```java
@PostMapping("/save")
public Result save(@RequestBody User user) {
    return Result.success(userService.save(user));
}
```

### 常用配置

```java
// 1. 可选请求体
@PostMapping("/save")
public Result save(@RequestBody(required = false) User user) {
    if (user == null) {
        return Result.error("请求体不能为空");
    }
    return Result.success(userService.save(user));
}

// 2. 使用 String 接收原始 JSON
@PostMapping("/raw")
public Result raw(@RequestBody String json) {
    return Result.success(JSON.parseObject(json));
}
```

## 易错点/踩坑

- ❌ `@RequestBody` 用于 GET 请求 → GET 请求通常没有请求体
- ❌ 请求体格式与 Java 对象不匹配 → 400 Bad Request
- ❌ 缺少 Jackson 依赖 → 无法处理 JSON
- ❌ 中文乱码 → 检查 HttpMessageConverter 编码配置

## 代码示例

### 接收复杂对象

```java
public class OrderRequest {
    private Long userId;
    private List<OrderItem> items;
    private BigDecimal totalAmount;
    // getters/setters
}

@PostMapping("/create")
public Result createOrder(@RequestBody OrderRequest request) {
    return Result.success(orderService.create(request));
}
```

### 接收 Map

```java
@PostMapping("/save")
public Result save(@RequestBody Map<String, Object> data) {
    String name = (String) data.get("name");
    Integer age = (Integer) data.get("age");
    return Result.success();
}
```

### 全局配置日期格式

```java
@Configuration
public class JacksonConfig {
    @Bean
    public ObjectMapper objectMapper() {
        ObjectMapper mapper = new ObjectMapper();
        mapper.setDateFormat(new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));
        mapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
        return mapper;
    }
}
```

## 关联知识点

- `05_HttpMessageConverters自动配置`：消息转换原理
- `12_@ResponseBody响应体`：响应体的序列化
