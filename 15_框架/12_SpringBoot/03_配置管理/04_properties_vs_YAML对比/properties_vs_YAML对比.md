# properties vs YAML 对比

## 先说结论

| 场景 | 推荐格式 |
|------|----------|
| 配置简单、项少 | properties |
| 配置复杂、层级深 | YAML |
| 团队统一规范 | 按团队习惯 |
| 需要中文注释 | YAML |

## 深度解析

### 对比表

| 特性 | properties | YAML |
|------|-----------|------|
| 语法风格 | `key=value` | 缩进层级 |
| 层级表示 | `a.b.c=value` | 缩进 |
| 中文支持 | 需转义 | 直接写 |
| 多文档 | 不支持 | `---` 分隔 |
| 数组写法 | `arr[0]=x` | `- item` |
| Map 写法 | `map.key=value` | `key: value` |
| 文件后缀 | `.properties` | `.yml` / `.yaml` |
| 学习成本 | 低 | 稍高 |

### properties 示例

```properties
# 层级用点
server.servlet.context-path=/api
server.port=8080

# 数组用索引
server.address[0]=127.0.0.1
server.address[1]=192.168.1.1

# 中文需转义
app.title=\u6211\u7684\u5E94\u7528
```

### YAML 示例

```yaml
# 层级用缩进
server:
  servlet:
    context-path: /api
  port: 8080

# 数组用短横线
server:
  address:
    - 127.0.0.1
    - 192.168.1.1

# 中文直接写
app:
  title: 我的应用
```

### 互相转换

```properties
# properties
server.port=8080
server.servlet.context-path=/api
spring.datasource.url=jdbc:mysql://localhost:3306/test
```

```yaml
# 等价 YAML
server:
  port: 8080
  servlet:
    context-path: /api
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/test
```

## 选择建议

### 选 properties
- 配置项少且扁平
- 团队习惯 properties
- 需要兼容旧项目

### 选 YAML
- 配置项多、层级深
- 需要多文档
- 配置文件需要人类可读性好

## 关联知识点

- [[01_application_properties格式]] — properties 详解
- [[02_application_yaml格式]] — YAML 详解
- [[14_SpringBoot配置优先级]] — 两者加载顺序
