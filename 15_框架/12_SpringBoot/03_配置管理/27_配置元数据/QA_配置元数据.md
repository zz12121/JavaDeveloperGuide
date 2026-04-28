# 配置元数据 QA

## Q1：如何启用配置元数据？

**A**：添加依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-configuration-processor</artifactId>
    <optional>true</optional>
</dependency>
```

然后重新编译项目。

---

## Q2：IDE 提示不显示怎么办？

**A**：检查：
1. 依赖是否添加
2. 是否重新编译
3. IDE 是否启用注解处理器
4. metadata 文件是否存在

---

## Q3：自定义属性能生成元数据吗？

**A**：能。使用 `@ConfigurationProperties` 的类会自动生成。

---

## Q4：可以手动添加提示吗？

**A**：可以：

```java
@ConfigurationProperties(prefix = "app")
@Metadata(defaultValue = "default mode", description = "Application mode")
public class AppProperties {
    @Metadata(since = "1.5.0", description = "Custom property")
    private String customProp;
}
```

---

## Q5：元数据文件在哪里查看？

**A**：编译后生成在：
```
target/classes/META-INF/spring-configuration-metadata.json
```

---

## Q6：为什么元数据提示不完整？

**A**：可能是：
- 使用 `@Value` 而不是 `@ConfigurationProperties`
- 未正确使用注解处理器
- 编译缓存问题，clean + rebuild
