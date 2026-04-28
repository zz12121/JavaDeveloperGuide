# application.properties 格式

## 先说结论

Spring Boot 默认配置文件，简单的键值对格式，适合配置项不多、层级简单的场景。

## 深度解析

### 基础语法

```properties
# 注释以 # 开头
server.port=8080                    # 键=值
server.servlet.context-path=/app    # 点分隔层级
spring.datasource.url=jdbc:mysql://localhost:3306/test
```

### 特点

| 特性 | 说明 |
|------|------|
| 编码 | 默认 ISO-8859-1，中文需转义 `\uXXXX` |
| 大小写 | 不敏感，`serverPort` 和 `SERVERPORT` 等价 |
| 层级表示 | 用点 `.` 连接，如 `server.servlet.context-path` |
| 数组表示 | 用索引 `[0]`，如 `server.address[0]=127.0.0.1` |

### 常见配置项

```properties
# 服务器
server.port=8080
server.servlet.context-path=/api

# 日志
logging.level.root=INFO
logging.level.com.example=DEBUG

# 数据源
spring.datasource.url=jdbc:mysql://localhost:3306/db
spring.datasource.username=root
spring.datasource.password=123456

# JPA
spring.jpa.show-sql=true
spring.jpa.hibernate.ddl-auto=update
```

### 中文转义

properties 文件中文需要 Unicode 转义：

```properties
# 原始中文
app.title=我的应用

# 转义后
app.title=\u6211\u7684\u5E94\u7528
```

## 易错点/踩坑

- ❌ `server: port: 8080` — properties 用 `=` 不是 `:`
- ❌ `port : 8080` — 键中不要加空格
- ❌ 中文字符直接写 — 要转义或改用 YAML
- ❌ 重复配置 — 后面的会覆盖前面的（按优先级）

## 关联知识点

- [[02_application_yaml格式]] — YAML 格式对比
- [[04_properties_vs_YAML对比]] — 两种格式选择
- [[14_SpringBoot配置优先级]] — 配置覆盖规则
