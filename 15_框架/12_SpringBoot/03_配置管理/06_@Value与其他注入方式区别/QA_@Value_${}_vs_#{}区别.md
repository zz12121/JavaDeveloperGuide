# @Value ${} vs #{} 区别 QA

## Q1：${} 和 #{} 可以同时用吗？

**A**：可以，常见用法是 `#{}` 中嵌套 `${}`：

```java
@Value("#{'${app.db.driver}'.trim()}")
private String driver;
```

---

## Q2：为什么用 ${} 而不是直接硬编码值？

**A**：
- **可配置**：不重新编译就能修改
- **多环境**：dev/prod 使用不同配置
- **可测试**：单元测试时可以注入 mock 值

---

## Q3：#{} 能读取配置文件吗？

**A**：不能直接读取，需要嵌套 `${}`：

```java
// 错误
@Value("#{'server.port'}")  // 读 literal "server.port"

// 正确
@Value("#{'${server.port}'}")  // 嵌套方式
```

---

## Q4：${} 支持默认值，那 #{} 支持吗？

**A**：SpEL 原生不支持默认值，但可以配合 `${}` 实现：

```java
@Value("${app.max:#{100}}")
private int max;  // ${} 提供默认值 100
```

---

## Q5：哪个性能更好？

**A**：**${}** 性能更好，因为只是简单字符串替换。`#{}` 需要解析和执行 SpEL 表达式。

---

## Q6：${} 读取的值是 String，注入 int/boolean 怎么转？

**A**：Spring 会自动类型转换：

```java
@Value("${app.port}")        // String → int
private int port;

@Value("${app.enabled:true}") // String → boolean
private boolean enabled;
```
