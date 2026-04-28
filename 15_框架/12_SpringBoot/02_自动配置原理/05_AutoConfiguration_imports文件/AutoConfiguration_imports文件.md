# AutoConfiguration.imports 文件

## 先说结论

`AutoConfiguration.imports` 是 Spring Boot 2.7+ 推荐的自动配置声明文件，位于 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`，每行声明一个自动配置类的全限定名，替代了旧的 `spring.factories` 格式。

## 深度解析

### 文件位置

```
META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

### 文件格式

```text
# 每行一个自动配置类，支持 # 注释
# 注释行以 # 开头

# Web 自动配置
org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration

# 数据源自动配置
org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration

# 自定义自动配置
com.example.mystarter.MyAutoConfiguration
```

### 对比 spring.factories

| 对比项 | spring.factories | AutoConfiguration.imports |
|--------|------------------|--------------------------|
| 路径 | `META-INF/spring.factories` | `META-INF/spring/xxx.imports` |
| 格式 | Properties 键值对 | 纯文本，每行一个类 |
| key 名称 | `EnableAutoConfiguration=...` | 无需 key，直接列表 |
| 续行符 | `\` | 不需要 |
| 注释 | 不支持 | 支持 `#` 注释 |
| 加载性能 | 较慢 | 更快 |
| Spring Boot 版本 | 2.7 之前 | **2.7+ 推荐** |

### Spring Boot 官方配置示例

Spring Boot 官方的 `spring-boot-autoconfigure.jar` 中：
```
META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

部分内容：
```text
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration
org.springframework.boot.autoconfigure.data.jdbc.JdbcRepositoriesAutoConfiguration
org.springframework.boot.autoconfigure.data.jpa.JpaRepositoriesAutoConfiguration
org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration
org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
org.springframework.boot.autoconfigure.jdbc.DataSourceTransactionManagerAutoConfiguration
org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaAutoConfiguration
org.springframework.boot.autoconfigure.web.servlet.DispatcherServletAutoConfiguration
org.springframework.boot.autoconfigure.web.servlet.ServletWebServerFactoryAutoConfiguration
org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration
```

## 易错点/踩坑

- ❌ **文件路径写错** — 必须是 `META-INF/spring/`，不是 `META-INF/` 直接下
- ❌ **目录名大小写** — Windows 不区分，Linux 区分，确保 `spring` 小写
- ❌ **文件名拼写** — `AutoConfiguration.imports`，不是 `autoconfiguration.imports`

## 关联知识点

- [[spring.factories文件]] — 旧版格式对比
- [[AutoConfigurationImportSelector]] — 读取此文件的调用方
- [[SpringBoot2_7自动配置变化]] — 变更详情
