# ConditionEvaluationReport

## Q1：ConditionEvaluationReport 有什么用？

**A**：主要用于自定义诊断和日志记录：

```java
// 在应用启动后打印配置诊断信息
@Autowired
private BeanFactory beanFactory;

@PostConstruct
public void printDiagnostics() {
    ConditionEvaluationReport report =
        ConditionEvaluationReport.get(beanFactory);

    log.info("=== Auto Configuration Report ===");
    log.info("Loaded: {}", report.getPositiveMatches().size());
    log.info("Skipped: {}", report.getNegativeMatches().size());
}
```

---

## Q2：如何在启动失败时获取报告？

**A**：

```java
// 在 FailureAnalyzer 中使用
public class MyFailureAnalyzer implements FailureAnalyzer {

    @Override
    public AnalysisResult analyze(Throwable failure) {
        if (failure.getCause() instanceof BeanCreationException) {
            // 获取导致失败的 Bean 名称
            String beanName = ((BeanCreationException) failure.getCause())
                .getBeanName();

            // 尝试从 BeanDefinitionRegistry 获取报告
            // 注意：此时应用上下文可能尚未完全创建
        }
        return null;
    }
}
```

---

## Q3：如何在日志中输出完整报告？

**A**：

```properties
logging:
  level:
    org.springframework.boot.autoconfigure.condition: DEBUG
```

这样会输出所有条件判断的详细日志，包括每个 `@ConditionalOn*` 的判断过程。

---

## Q4：getConditionAndOutcomesBySource 是什么？

**A**：按配置类来源分组查看条件评估结果：

```java
report.getConditionAndOutcomesBySource().forEach((source, outcomes) -> {
    log.info("Source: {}", source);
    outcomes.getConditions().forEach(condition -> {
        log.info("  - {}: {}", condition.getClass().getSimpleName(), outcome);
    });
});
```

用于分析某个特定配置类（如 `WebMvcAutoConfiguration`）的条件评估详情。

