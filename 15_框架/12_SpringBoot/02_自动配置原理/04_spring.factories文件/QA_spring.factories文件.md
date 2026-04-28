# spring.factories 文件

## Q1：spring.factories 文件的工作原理是什么？

**A**：`SpringFactoriesLoader` 类负责读取：

1. 扫描所有 JAR 中 `META-INF/spring.factories` 文件
2. 按 Properties 格式解析，按 `key=value` 读取
3. 对 `EnableAutoConfiguration` key，返回逗号分隔的配置类全限定名列表
4. 合并多个 JAR 中的配置，重复项去重

```java
// 源码核心
List<String> configurations = new ArrayList<>();
for (UrlResource resource : resources) {
    Properties properties = PropertiesLoaderUtils.loadProperties(resource);
    String value = properties.getProperty(factoryType.getName());
    if (value != null) {
        // 逗号分隔
        for (String name : StringUtils.commaDelimitedListToStringArray(value)) {
            configurations.add(name.trim());
        }
    }
}
```

---

## Q2：spring.factories 可以声明多个 key 吗？

**A**：可以，一个文件中可以声明多个不同 key：

```properties
# org.springframework.boot.autoconfigure.EnableAutoConfiguration
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  com.example.MyAutoConfiguration

# org.springframework.context.ApplicationContextInitializer
org.springframework.context.ApplicationContextInitializer=\
  com.example.MyContextInitializer

# org.springframework.boot.SpringBootExceptionReporter
org.springframework.boot.SpringBootExceptionReporter=\
  com.example.MyFailureAnalyzer
```

---

## Q3：多个 JAR 中有同名 key 怎么办？

**A**：分两种情况：

| 情况 | 结果 |
|------|------|
| `EnableAutoConfiguration` key | **追加合并**（多个 JAR 的配置都生效） |
| 其他 key（如自定义） | **后者覆盖前者**（取决于 classLoader 实现） |

```properties
# JAR A 中
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  com.example.AutoConfigA

# JAR B 中
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  com.example.AutoConfigB

# 最终：两个配置都会加载
```

**注意**：只有 `EnableAutoConfiguration` 会合并，其他 key 行为不确定。

---

## Q4：如何排查 spring.factories 配置不生效？

**A**：

1. **检查文件路径**：`META-INF/spring.factories`（不是 `resources/META-INF/`）
2. **检查 key 拼写**：`org.springframework.boot.autoconfigure.EnableAutoConfiguration`
3. **开启 debug 日志**：`logging.level.org.springframework.boot.autoconfigure=DEBUG`
4. **检查 classpath 顺序**：后加载的 JAR 可能覆盖配置

---

## Q5：为什么推荐迁移到 AutoConfiguration.imports？

**A**：迁移原因：

| 对比 | spring.factories | AutoConfiguration.imports |
|------|------------------|--------------------------|
| 格式 | Properties（键值对） | 纯文本（每行一个类） |
| 语法 | 需转义、续行符 | 更简洁直观 |
| 加载速度 | 需解析 Properties | 更快速 |
| Spring Boot 推荐 | 废弃 | **推荐** |
| IDE 支持 | 差 | 好 |

