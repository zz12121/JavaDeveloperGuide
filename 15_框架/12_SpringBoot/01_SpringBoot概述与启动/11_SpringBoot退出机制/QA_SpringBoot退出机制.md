# Spring Boot 退出机制

## Q1：Spring Boot 的优雅关闭是什么？

**A**：优雅关闭（Graceful Shutdown）确保在应用停止前，完成所有正在处理的请求。

```yaml
server:
  shutdown: graceful  # 优雅关闭（默认）
```

**关闭过程**：
```
1. 停止接受新请求
2. 等待正在处理的请求完成
3. 释放资源
4. 退出进程
```

**立即关闭**：
```yaml
server:
  shutdown: immediate  # 立即停止
```

---

## Q2：如何配置优雅关闭超时时间？

**A**：

```yaml
server:
  shutdown: graceful

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s  # 默认 30 秒
```

**超时后**：会强制退出，可能中断请求。

---

## Q3：@PreDestroy 和 DisposableBean.destroy() 的区别？

**A**：

| 对比 | @PreDestroy | DisposableBean.destroy() |
|------|-------------|--------------------------|
| **来源** | JSR-250 | Spring Bean 生命周期 |
| **实现方式** | 注解方法 | 接口实现 |
| **执行顺序** | 先 | 后 |

**@PreDestroy 示例**：
```java
@Component
public class CacheManager {
    
    private Map<String, Object> cache = new HashMap<>();
    
    @PreDestroy
    public void clear() {
        cache.clear();
    }
}
```

**DisposableBean 示例**：
```java
@Component
public class ConnectionManager implements DisposableBean {
    
    private Connection connection;
    
    @Override
    public void destroy() throws Exception {
        if (connection != null) {
            connection.close();
        }
    }
}
```

---

## Q4：如何主动关闭 Spring Boot 应用？

**A**：

**方式一：通过 ApplicationContext**
```java
@Autowired
private ConfigurableApplicationContext ctx;

public void shutdown() {
    ctx.close();
}
```

**方式二：通过 System.exit()**
```java
public static void main(String[] args) {
    ConfigurableApplicationContext ctx = SpringApplication.run(App.class, args);
    
    // 某个条件下关闭
    if (shouldStop()) {
        SpringApplication.exit(ctx, () -> 0);  // 退出码 0
    }
}
```

**方式三：通过 Actuator 端点**
```bash
curl -X POST http://localhost:8080/actuator/shutdown
```

---

## Q5：Spring Boot 退出码有什么意义？

**A**：退出码用于外部进程判断应用状态：

| 退出码 | 含义 |
|--------|------|
| 0 | 正常退出 |
| 1 | 异常退出（启动失败或运行时异常） |
| 非零 | 错误退出 |

**自定义退出码**：
```java
public static void main(String[] args) {
    System.exit(SpringApplication.exit(
        SpringApplication.run(App.class, args),
        () -> 42  // 自定义退出码
    ));
}
```

---

## Q6：关闭钩子（Shutdown Hook）是什么？

**A**：JVM 关闭时自动执行的代码。

```java
@SpringBootApplication
public class Application {
    
    public static void main(String[] args) {
        ConfigurableApplicationContext ctx = 
            SpringApplication.run(Application.class, args);
        
        // 注册关闭钩子
        ctx.registerShutdownHook();
    }
}
```

**触发场景**：
- `kill -15 SIGTERM`（推荐）
- Ctrl+C（SIGINT）
- JVM 异常退出

**不触发场景**：
- `kill -9 SIGKILL`（强制杀死）

---

## Q7：Docker/K8s 环境下如何优雅关闭？

**A**：

**Dockerfile**：
```dockerfile
# 允许长时间运行的请求
STOPSIGNAL SIGTERM
```

**K8s 配置**：
```yaml
spec:
  terminationGracePeriodSeconds: 60  # 等待时间
  containers:
  - name: app
    lifecycle:
      preStop:
        exec:
          command: ["sleep", "5"]  # 等待 5 秒后再开始关闭
```

**Spring Boot 配置**：
```yaml
server:
  shutdown: graceful

spring:
  lifecycle:
    timeout-per-shutdown-phase: 55s  # 小于 K8s 的 60s
```
