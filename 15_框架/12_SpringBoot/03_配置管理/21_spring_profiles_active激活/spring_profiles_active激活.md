# spring.profiles.active 激活

## 先说结论

`spring.profiles.active` 是最常用的 profile 激活方式，可以在配置文件中设置或通过命令行/环境变量传入。

## 深度解析

### 配置文件方式

```yaml
# application.yml
spring:
  profiles:
    active: dev
```

### 多 profile 激活

```yaml
spring:
  profiles:
    active: dev,common
```

### 命令行方式（推荐）

```bash
java -jar app.jar --spring.profiles.active=prod
```

### 环境变量方式

```bash
# Linux/Mac
export SPRING_PROFILES_ACTIVE=prod

# Windows
set SPRING_PROFILES_ACTIVE=prod

# Docker
docker run -e SPRING_PROFILES_ACTIVE=prod myapp
```

### spring.profiles.include

```yaml
# 在激活 dev 的同时包含 common 配置
spring:
  profiles:
    active: dev
    include: common
```

### spring.profiles.add

```yaml
# 2.4+ 替代 include
spring:
  profiles:
    active: dev
    add: common
```

## 易错点/踩坑

- ❌ `spring.profiles.active` 写在 profile 文件中 — 可能形成循环引用
- ❌ 多 profile 加载顺序不确定 — 可能导致配置覆盖问题
- ❌ 环境变量覆盖配置文件 — 可能导致本地调试困惑

## 关联知识点

- [[19_多环境配置]] — 环境配置理念
- [[20_@Profile激活环境]] — @Profile 注解
- [[22_application_profile配置文件]] — profile 文件规范
