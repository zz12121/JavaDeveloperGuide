# Spring Boot 退出机制

## 先说结论

Spring Boot 应用通过**优雅关闭**（Graceful Shutdown）和**退出码**机制确保应用安全停止，释放资源、处理在途请求。

## 深度解析

### 退出流程

```
┌─────────────────────────────────────────────────────────────┐
│                      退出流程                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   kill -TERM / Ctrl+C                                      │
│         │                                                  │
│         ↓                                                  │
│   SpringApplication 接收到关闭信号                          │
│         │                                                  │
│         ├── 发布 ApplicationShutdownEvent                  │
│         ├── 停止 /actuator/shutdown 端点（如果启用）        │
│         │                                                  │
│         ├── 停止 Web 服务器                                │
│         │    │                                             │
│         │    ├── 停止接受新请求                             │
│         │    └── 等待在途请求完成（超时时间可配置）          │
│         │                                                  │
│         ├── 调用 @PreDestroy 方法                          │
│         │                                                  │
│         ├── 调用 DisposableBean.destroy()                 │
│         │                                                  │
│         ├── 关闭 DataSource 连接池                          │
│         │                                                  │
│         └── 关闭 JVM                                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 优雅关闭配置

```yaml
server:
  shutdown: graceful  # 默认 graceful

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s  # 等待超时，默认 30 秒
```

### 关闭模式

| 模式 | 值 | 行为 |
|------|------|------|
| **优雅关闭** | `graceful` | 等待在途请求完成 |
| **立即关闭** | `immediate` | 立即停止所有请求 |

### 退出码

| 退出码 | 说明 |
|--------|------|
| 0 | 正常退出 |
| 1 | 异常退出 |
| 130 | Ctrl+C 退出（SIGINT） |

## 易错点/踩坑

- ❌ 忽视优雅关闭，导致请求中断
- ❌ 超时时间设置过短，请求未处理完
- ❌ @PreDestroy 耗时过长阻塞关闭

## 代码示例

### 配置优雅关闭

```yaml
server:
  shutdown: graceful

spring:
  lifecycle:
    timeout-per-shutdown-phase: 60s  # 等待 60 秒
```

### @PreDestroy 清理资源

```java
@Component
public class ResourceCleanup {
    
    @PreDestroy
    public void cleanup() {
        // 关闭连接、清理缓存等
        System.out.println("资源清理中...");
    }
}
```

### DisposableBean 释放资源

```java
@Component
public class MyService implements DisposableBean {
    
    private Connection connection;
    
    @Override
    public void destroy() throws Exception {
        if (connection != null) {
            connection.close();
        }
    }
}
```

### 关闭钩子

```java
@SpringBootApplication
public class Application {
    
    public static void main(String[] args) {
        ConfigurableApplicationContext ctx = SpringApplication.run(Application.class, args);
        
        // 注册关闭钩子
        ctx.registerShutdownHook();
        
        // 或手动关闭
        // ctx.close();
    }
}
```

## 关联知识点

- `@PreDestroy` 注解
- DisposableBean 接口
- 优雅关闭配置
- Spring Boot Actuator shutdown 端点
