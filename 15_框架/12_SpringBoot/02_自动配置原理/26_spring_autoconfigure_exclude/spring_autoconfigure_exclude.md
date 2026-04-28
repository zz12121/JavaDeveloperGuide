# spring.autoconfigure.exclude 排除配置

## 先说结论

`spring.autoconfigure.exclude` 是 Spring Boot 配置文件中的属性，用于排除指定的自动配置类。相比注解方式，配置文件更灵活，无需重新编译代码即可修改。

## 深度解析

### 配置方式

```properties
# application.properties
spring.autoconfigure.exclude=\
  org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration,\
  org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration
```

```yaml
# application.yml（推荐）
spring:
  autoconfigure:
    exclude:
      - org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
      - org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration
```

### 配置位置

| 环境 | 配置方式 |
|------|----------|
| 开发环境 | `src/main/resources/application.yml` |
| 测试环境 | `@ActiveProfiles` 指定 |
| 生产环境 | 环境变量 / 命令行参数 |

### 优先级

```
命令行参数 > 环境变量 > application.yml > application.properties
```

```bash
# 命令行覆盖配置文件
java -jar app.jar --spring.autoconfigure.exclude=com.example.MyAutoConfiguration
```

## 关联知识点

- [[自动配置排除详解]] — 排除方式汇总
- [[多环境配置]] — 环境差异化配置
