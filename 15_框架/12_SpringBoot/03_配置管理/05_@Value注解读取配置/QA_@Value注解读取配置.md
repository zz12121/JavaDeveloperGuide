# @Value 注解读取配置 QA

## Q1：@Value 读取的配置是静态的还是动态的？

**A**：是启动时加载一次的静态值。如果需要动态更新，配合 `@RefreshScope` 使用。

---

## Q2：配置不存在时不想报错怎么办？

**A**：两种方式：

```java
// 方式一：使用默认值
@Value("${app.timeout:5000}")
private int timeout;

// 方式二：使用 SpEL 判断
@Value("#{ '${app.timeout}'.isEmpty() ? 5000 : '${app.timeout}' }")
private int timeout;
```

---

## Q3：@Value 能注入到静态字段吗？

**A**：不能直接注入。正确方式：

```java
@Component
public class AppConfig {
    private static String name;

    @Value("${app.name}")
    public void setName(String name) {
        AppConfig.name = name;  // 通过方法注入
    }

    public static String getName() {
        return name;
    }
}
```

---

## Q4：@Value 读取 Map 和 List 怎么写？

**A**：

```java
// List（逗号分隔）
@Value("${app.servers:localhost,127.0.0.1}")
private List<String> servers;

// Map（YAML 格式）
@Value("#{${app.map}}")
private Map<String, String> map;
```

```yaml
app:
  servers: localhost,127.0.0.1
  map: "{key1: 'value1', key2: 'value2'}"
```

---

## Q5：@Value 和 @ConfigurationProperties 能同时用吗？

**A**：可以，但没必要。通常选择一种：
- 单个值用 `@Value`
- 配置类用 `@ConfigurationProperties`

---

## Q6：@Value 注入的字段能加 @Autowired 吗？

**A**：不需要。`@Value` 本身就是 Spring 的依赖注入机制，无需额外加 `@Autowired`。
