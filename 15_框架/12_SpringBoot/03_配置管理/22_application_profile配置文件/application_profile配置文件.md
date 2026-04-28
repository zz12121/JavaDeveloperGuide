# application-{profile}.properties/yml

## 先说结论

profile 配置文件是 Spring Boot 多环境配置的标准方式，命名格式为 `application-{profile}.yml` 或 `application-{profile}.properties`。

## 深度解析

### 文件命名规范

```
application.yml              # 公共配置
application-dev.yml          # 开发环境
application-test.yml         # 测试环境
application-prod.yml         # 生产环境
application-staging.yml      # 预发布环境
```

### 加载顺序

```
1. application.yml（最先加载）
2. application-{profile}.yml（按 spring.profiles.active）
3. 命令行参数（最后加载，最高优先级）
```

### 完整示例

**application.yml**
```yaml
server:
  port: 8080
spring:
  application:
    name: myapp
```

**application-dev.yml**
```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/dev_db
    password: dev
logging:
  level:
    root: DEBUG
```

**application-prod.yml**
```yaml
spring:
  datasource:
    url: jdbc:mysql://prod-server:3306/prod_db
    password: ${DB_PASSWORD}
logging:
  level:
    root: WARN
```

### properties 格式

```properties
# application-dev.properties
server.port=8080
spring.datasource.url=jdbc:mysql://localhost:3306/dev_db
```

## 易错点/踩坑

- ❌ profile 文件名拼写错误 — `application-prod.yml` 不能写成 `application-product.yml`
- ❌ 只配 profile 文件不激活 — 需要设置 `spring.profiles.active`
- ❌ profile 配置覆盖不了公共配置 — 按优先级，后加载的覆盖先加载的

## 关联知识点

- [[19_多环境配置]] — 环境配置理念
- [[20_@Profile激活环境]] — @Profile 注解
- [[21_spring_profiles_active激活]] — 激活方式
