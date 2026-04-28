# Spring Boot 2.7 自动配置变化

## Q1：Spring Boot 2.7 自动配置最大变化是什么？

**A**：两个核心变化：

**1. 配置文件格式变更**
```properties
# 旧：spring.factories
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  com.example.MyAutoConfiguration

# 新：AutoConfiguration.imports
com.example.MyAutoConfiguration
```

**2. 注解变更**
```java
// 旧
@Configuration
@ConditionalOnClass(Xxx.class)
public class MyAutoConfiguration { }

// 新
@AutoConfiguration
@ConditionalOnClass(Xxx.class)
public class MyAutoConfiguration { }
```

---

## Q2：@AutoConfiguration 和 @Configuration 有什么区别？

**A**：

| 对比项 | @Configuration | @AutoConfiguration |
|--------|---------------|-------------------|
| 用途 | 普通配置类 | 自动配置类 |
| 索引支持 | 不支持 | 支持 `@Indexed` 加速 |
| 语义 | 业务配置 | 框架预置配置 |
| Spring Boot 版本 | 1.x+ | 2.4+（推荐） |
| 相互关系 | 包含关系（@AutoConfiguration 内部使用 @Configuration） |

---

## Q3：2.7 之前的老项目需要迁移吗？

**A**：建议迁移，但不强制：

| 情况 | 建议 |
|------|------|
| 新项目 | 直接使用新格式 |
| 现有项目 | 可以暂时保留旧格式，但尽快迁移 |
| Spring Boot 3.x | 可能完全移除对 spring.factories 的支持 |

---

## Q4：如何判断项目是否使用了旧格式？

**A**：

```bash
# 检查是否存在 spring.factories 中的自动配置声明
grep -r "EnableAutoConfiguration" src/main/resources/META-INF/
```

或者：启动应用时，Spring Boot 会输出类似警告：
```
SpringFactoriesLoader not supported with AutoConfiguration.imports - spring.factories is deprecated
```

---

## Q5：迁移过程中有哪些常见错误？

**A**：

| 错误 | 原因 |
|------|------|
| 路径写错 | 必须是 `META-INF/spring/`，不是 `META-INF/` |
| 文件名写错 | `AutoConfiguration.imports`（不是 `autoconfiguration.imports`） |
| 内容格式错误 | 新格式不需要 key，直接每行一个类名 |
| 重复配置 | imports 和 spring.factories 都写了同一配置 |

