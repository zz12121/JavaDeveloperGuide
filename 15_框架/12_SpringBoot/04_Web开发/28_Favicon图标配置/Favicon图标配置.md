# Favicon图标配置

## 先说结论

Spring Boot 自动处理 Favicon，文件放在 `static/` 目录下名为 `favicon.ico` 即可。

## 深度解析

### 放置位置

```
src/main/resources/static/
└── favicon.ico
```

### 自定义 Favicon

```yaml
spring:
  mvc:
    favicon:
      enabled: true
```

### 禁用 Favicon

```yaml
spring:
  mvc:
    favicon:
      enabled: false
```

## 关联知识点

- `25_静态资源访问规则`：静态资源基础
