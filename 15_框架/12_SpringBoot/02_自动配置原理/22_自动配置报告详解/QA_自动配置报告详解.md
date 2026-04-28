# 自动配置报告详解

## Q1：debug: true 和 Actuator 端点有什么区别？

**A**：

| 对比项 | debug: true | Actuator 端点 |
|--------|-------------|---------------|
| 输出位置 | 控制台启动日志 | HTTP 接口 |
| 实时性 | 仅启动时 | 运行时也可查看 |
| 格式 | 文本日志 | JSON 结构化 |
| 适合场景 | 开发调试 | 生产监控 |
| 性能影响 | 无 | 极小 |

---

## Q2：报告中 positiveMatches 和 negativeMatches 是什么意思？

**A**：

| 类型 | 含义 | 是否需要关注 |
|------|------|-------------|
| `positiveMatches` | 条件满足，配置已加载 | 通常不需要 |
| `negativeMatches` | 条件不满足，配置未加载 | **需要关注** |

`negativeMatches` 不一定是问题——Spring Boot 设计就是按需加载，没有 Redis 依赖就不加载 Redis 配置是正常行为。

---

## Q3：如何找到导致配置不生效的具体原因？

**A**：查看 `negativeMatches` 中的 `reason` 字段：

```json
{
  "negativeMatches": {
    "org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration": {
      "notMatched": [
        {
          "condition": "OnClassCondition",
          "message": "@ConditionalOnClass did not find required class 'org.springframework.data.redis.core.RedisOperations'"
        }
      ],
      "matched": []
    }
  }
}
```

**原因**：classpath 中没有 `spring-data-redis` 依赖。

---

## Q4：如何用代码获取配置报告？

**A**：

```java
@Autowired
private BeanFactory beanFactory;

@PostConstruct
public void printReport() {
    ConditionEvaluationReport report =
        ConditionEvaluationReport.get(beanFactory);

    System.out.println("=== Positive Matches ===");
    report.getPositiveMatches().forEach((k, v) -> System.out.println(k));

    System.out.println("=== Negative Matches ===");
    report.getNegativeMatches().forEach((k, v) -> {
        System.out.println(k + " - " + v.getCondition().toString());
    });
}
```

---

## Q5：报告太长，如何过滤只看关键的？

**A**：

```bash
# Linux/Mac
java -jar app.jar --debug 2>&1 | grep -A5 "DataSource"

# Windows PowerShell
java -jar app.jar --debug 2>&1 | Select-String -Pattern "DataSource" -Context 0,5
```

