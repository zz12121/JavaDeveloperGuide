# ConditionEvaluationReport

## 先说结论

`ConditionEvaluationReport` 是 Spring Boot 提供的编程式 API，用于获取自动配置的条件评估结果。可在应用启动后查询哪些条件通过、哪些未通过，用于自定义诊断或日志记录。

## 深度解析

### 获取方式

```java
// 方式一：从 BeanFactory 获取
@Autowired
private BeanFactory beanFactory;

@PostConstruct
public void init() {
    ConditionEvaluationReport report =
        ConditionEvaluationReport.get(beanFactory);
}

// 方式二：从 ApplicationContext 获取
@Autowired
private ApplicationContext context;

@PostConstruct
public void init() {
    ConditionEvaluationReport report =
        ConditionEvaluationReport.get(context.getBeanFactory());
}
```

### 核心方法

| 方法 | 返回值 | 说明 |
|------|--------|------|
| `getPositiveMatches()` | `Map<String, ConditionEvaluationReport.Details>` | 条件通过的配置 |
| `getNegativeMatches()` | `Map<String, ConditionEvaluationReport.Details>` | 条件未通过的配置 |
| `getUnconditionalConditions()` | `List<ConditionAndOutcomes>` | 无条件配置 |
| `getConditionAndOutcomesBySource()` | `Map<String, ConditionAndOutcomes>` | 按来源分组 |

### 报告结构

```java
public class ConditionEvaluationReport.Details {
    private ConditionEvaluationReport.ConditionAndOutcomes conditionAndOutcomes;
    // 条件评估详情
}
```

### 日志输出示例

```java
@PostConstruct
public void logConditionReport() {
    ConditionEvaluationReport report =
        ConditionEvaluationReport.get(beanFactory);

    log.info("=== Positive Matches ===");
    report.getPositiveMatches().forEach((name, details) ->
        log.info("{}: {}", name, details)
    );

    log.info("=== Negative Matches ===");
    report.getNegativeMatches().forEach((name, details) ->
        log.info("{}: {}", name, details)
    );
}
```

## 关联知识点

- [[自动配置报告详解]] — 报告功能概述
- [[启动失败分析]] — 启动失败处理
