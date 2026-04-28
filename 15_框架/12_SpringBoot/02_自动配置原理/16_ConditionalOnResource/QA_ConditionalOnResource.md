# @ConditionalOnResource 条件判断

## Q1：resources 支持哪几种资源路径格式？

**A**：

| 格式 | 说明 | 示例 |
|------|------|------|
| `classpath:` | classpath 中的资源 | `classpath:config.xml` |
| `classpath*:` | 支持多个 jar 包中的同名资源 | `classpath*:META-INF/spring/*.xml` |
| `file:` | 文件系统绝对路径 | `file:/etc/myapp/config.properties` |
| 无前缀 | classpath 相对路径 | `META-INF/spring.xml` |

---

## Q2：多个 resources 是 AND 还是 OR 关系？

**A**：**AND 关系**——所有资源都存在才通过：

```java
@ConditionalOnResource(resources = {
    "classpath:ehcache.xml",
    "classpath:cache-config.xml"
})
// 必须两个文件都存在才加载
```

---

## Q3：@ConditionalOnResource 有什么实际应用？

**A**：

```java
// MyBatis 配置：如果存在 mybatis-config.xml，使用外部配置
@AutoConfiguration
@ConditionalOnResource(resources = "classpath:mybatis-config.xml")
public class MyBatisXmlConfigAutoConfiguration {
    // 加载外部 mybatis-config.xml
}
```

```java
// 数据库迁移：存在 db/migration/*.sql 时启用 Flyway
@AutoConfiguration
@ConditionalOnResource(resources = "classpath:db/migration/")
public class FlywayAutoConfiguration {
    // 启用 Flyway 自动化数据库迁移
}
```

---

## Q4：classpath: 和 classpath*: 有什么区别？

**A**：

| 格式 | 搜索范围 |
|------|----------|
| `classpath:config.xml` | 只在第一个匹配的 classpath 中查找 |
| `classpath*:config.xml` | 在所有 classpath（JAR 包）中查找 |

```java
// 假设有两个 JAR 包都有 spring.factories
classpath:spring.factories     // 只能找到第一个
classpath*:spring.factories    // 能找到所有
```

**注意**：`classpath*:` 会扫描所有 JAR，开销较大，慎用。

