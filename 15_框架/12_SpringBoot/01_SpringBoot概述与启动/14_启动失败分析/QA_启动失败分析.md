# 启动失败分析

## Q1：FailureAnalyzer 是什么？

**A**：FailureAnalyzer 是 Spring Boot 的**启动失败分析器**，将原始异常转换为用户友好的错误提示。

**对比**：

| 无 FailureAnalyzer | 有 FailureAnalyzer |
|-------------------|-------------------|
| `BeanCreationException: Error creating bean...` | **端口被占用** 8080 端口已被使用 |
| `Could not resolve placeholder 'xxx'...` | **配置缺失** 请添加 spring.datasource.url |

---

## Q2：Spring Boot 内置了哪些 FailureAnalyzer？

**A**：

| 分析器 | 场景 |
|--------|------|
| `PortInUseFailureAnalyzer` | 端口被占用 |
| `DataSourceBeanCreationFailureAnalyzer` | 数据源创建失败 |
| `NoSuchBeanDefinitionFailureAnalyzer` | Bean 不存在 |
| `BeanCurrentlyInCreationFailureAnalyzer` | 循环依赖 |
| `UnsatisfiedDependencyFailureAnalyzer` | 依赖不满足 |
| `NoUniqueBeanDefinitionFailureAnalyzer` | 多个候选 Bean |
| `ValidationConfigurationFailureAnalyzer` | 配置验证失败 |

---

## Q3：如何启用详细的失败分析？

**A**：

**方式一：debug 模式**
```yaml
debug: true
```

**方式二：日志级别**
```yaml
logging:
  level:
    org.springframework.boot.diagnostics: DEBUG
```

---

## Q4：如何自定义 FailureAnalyzer？

**A**：

**步骤一：实现接口**
```java
public class MyFailureAnalyzer implements FailureAnalyzer {
    
    @Override
    public FailureAnalysis analyze(Throwable cause) {
        if (cause instanceof MyCustomException) {
            return new FailureAnalysis(
                "错误标题",
                "错误描述和解决方案",
                cause
            );
        }
        return null;  // 不处理此异常
    }
}
```

**步骤二：注册**
```properties
# META-INF/spring.factories
org.springframework.boot.SpringApplicationFailureAnalyzer=\
  com.example.MyFailureAnalyzer
```

---

## Q5：FailureAnalyzer 的执行顺序？

**A**：

```
启动失败
    ↓
遍历所有 FailureAnalyzer
    ↓
第一个返回非 null 的 FailureAnalysis 被使用
    ↓
输出友好错误信息
```

**如果都不处理**：输出原始异常堆栈。

---

## Q6：常见启动失败场景及解决方案？

**A**：

| 错误 | 原因 | 解决方案 |
|------|------|----------|
| 端口被占用 | 8080 已被使用 | `server.port=8081` 或 kill 占用进程 |
| Bean 不存在 | 组件扫描失败 | 检查 @ComponentScan |
| 循环依赖 | A→B→A | 重构代码或 @Lazy 注入 |
| 配置缺失 | 缺少必需配置 | 添加到 application.yml |
| 数据源错误 | 数据库连接失败 | 检查 url/username/password |
| 类不匹配 | 版本冲突 | 排除冲突依赖 |

---

## Q7：如何调试启动失败？

**A**：

**1. 查看完整堆栈**
```yaml
# application.yml
logging:
  level:
    root: INFO
    com.example: DEBUG
```

**2. 禁用自动配置**
```java
@SpringBootApplication(exclude = {
    DataSourceAutoConfiguration.class  // 禁用数据源自动配置
})
```

**3. 跳过启动**
```java
// 只验证配置，不真正启动
SpringApplication app = new SpringApplication(App.class);
app.setWebApplicationType(WebApplicationType.NONE);
app.run(args);
```

**4. 使用 --dry-run（Spring Boot 2.6+）**
```bash
java -jar app.jar --spring.main.lazy-initialization=true
```
