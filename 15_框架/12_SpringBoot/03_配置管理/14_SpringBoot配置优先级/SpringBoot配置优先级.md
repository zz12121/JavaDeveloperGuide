# Spring Boot 配置优先级

## 先说结论

Spring Boot 支持 15 级配置源，后面的优先级高于前面。**命令行参数 > 环境变量 > application.properties/yml**。

## 深度解析

### 优先级列表（从低到高）

| 优先级 | 配置源 |
|--------|--------|
| 1 | Spring Boot 默认值 |
| 2 | classpath:application.properties |
| 3 | classpath:application.yml |
| 4 | classpath:application-{profile}.properties |
| 5 | classpath:application-{profile}.yml |
| 6 | file:./config/application.properties |
| 7 | file:./config/application.yml |
| 8 | file:./config/application-{profile}.properties |
| 9 | file:./config/application-{profile}.yml |
| 10 | file:./application.properties |
| 11 | file:./application.yml |
| 12 | file:./application-{profile}.properties |
| 13 | file:./application-{profile}.yml |
| 14 | 环境变量 |
| 15 | 命令行参数 `--xxx=yyy` |

### 目录结构

```
project/
├── config/
│   └── application.properties    # 优先级 6
├── application.properties        # 优先级 10
└── src/main/resources/
    └── application.properties    # 优先级 2
```

### 命令行参数

```bash
java -jar app.jar --server.port=9090
java -jar app.jar --spring.profiles.active=prod
```

### 环境变量

```bash
# Linux/Mac
export SERVER_PORT=9090

# Windows
set SERVER_PORT=9090
```

### 属性映射规则

| 属性 | 环境变量 |
|------|----------|
| `server.port` | `SERVER_PORT` |
| `server.servlet.context-path` | `SERVER_SERVLET_CONTEXT_PATH` |
| `app.config-file` | `APP_CONFIG_FILE` |

## 易错点/踩坑

- ❌ 本地配置覆盖测试环境 — 检查加载顺序
- ❌ 环境变量和 properties 冲突 — 环境变量优先级更高
- ❌ profile 配置没生效 — 检查激活方式

## 关联知识点

- [[16_命令行参数配置]] — 命令行参数详解
- [[17_环境变量自动映射]] — 环境变量转换规则
- [[22_application_profile配置文件]] — profile 配置加载
