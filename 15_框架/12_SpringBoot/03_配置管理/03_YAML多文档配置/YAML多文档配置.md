# YAML 多文档配置

## 先说结论

YAML 支持用 `---` 分隔多个文档，适合需要针对不同环境或条件加载不同配置的场景。

## 深度解析

### 基本语法

```yaml
# 第一个文档
server:
  port: 8080
---
# 第二个文档
server:
  port: 9090
---
# 第三个文档
server:
  port: 80
```

### 带 profile 的多文档

```yaml
# 默认配置
server:
  port: 8080
---
spring:
  config:
    activate:
      on-profile: dev
server:
  port: 8081
---
spring:
  config:
    activate:
      on-profile: prod
server:
  port: 8080
```

### 文档合并规则

同一键名：后面的文档覆盖前面的

```yaml
# 文档1
app:
  name: MyApp
  version: 1.0
---
# 文档2
app:
  version: 2.0
  author: 张三
```

最终结果：
```yaml
app:
  name: MyApp      # 文档1保留
  version: 2.0     # 文档2覆盖
  author: 张三     # 文档2新增
```

### 与 profile 的区别

| 特性 | 多文档 `---` | profile |
|------|-------------|---------|
| 加载条件 | 无条件依次加载 | 激活 profile 时加载 |
| 覆盖方式 | 键级别覆盖 | 文件级别覆盖 |
| 适用场景 | 默认配置分层 | 环境隔离 |

## 易错点/踩坑

- ❌ 多文档中重复键没生效 — 检查是否有语法错误或被覆盖
- ❌ `---` 前后有空格 — `---` 必须单独一行，前面不能有空格
- ❌ profile 文档和普通文档冲突 — 普通文档先加载，可能被覆盖

## 代码示例

```java
// 验证多文档加载
@SpringBootApplication
public class Demo {
    public static void main(String[] args) {
        ConfigurableApplicationContext ctx = SpringApplication.run(Demo.class, args);
        System.out.println(ctx.getEnvironment().getProperty("server.port"));
    }
}
```

## 关联知识点

- [[02_application_yaml格式]] — YAML 基础
- [[19_多环境配置]] — 环境配置理念
- [[23_spring_config_activate_on_profile]] — profile 激活条件
