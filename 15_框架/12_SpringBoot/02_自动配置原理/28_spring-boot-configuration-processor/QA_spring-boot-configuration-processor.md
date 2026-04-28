# spring-boot-configuration-processor

## Q1：spring-boot-configuration-processor 有什么用？

**A**：编译时注解处理器，生成配置元数据文件。

**核心价值**：
1. IDE 智能提示（配置属性名、类型、默认值）
2. 枚举值下拉提示
3. 配置文档自动生成

---

## Q2：为什么必须设置为 optional？

**A**：避免打包到生产环境。

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-configuration-processor</artifactId>
    <optional>true</optional>  ← 必须标记为 optional
</dependency>
```

**原因**：
- 运行时不需要此依赖
- 减少最终 JAR 大小
- 避免依赖冲突

---

## Q3：如何使用它生成配置属性提示？

**A**：在 `@ConfigurationProperties` 类上添加 Javadoc 描述。

```java
/**
 * 功能开关配置
 */
@ConfigurationProperties(prefix = "myapp.feature")
public class MyProperties {
    
    /**
     * 是否启用该功能
     */
    private boolean enabled = true;
}
```

生成的元数据会包含 description 字段，IDE 显示为注释提示。

---

## Q4：为什么不自动更新元数据？

**A**：因为元数据是在**编译时**生成的，不是运行时。

**解决方法**：
```bash
# 重新编译
mvn clean compile

# 或强制重新处理
mvn clean package -DskipTests
```

---

## Q5：Gradle 如何配置？

**A**：使用 annotationProcessor 配置。

```groovy
dependencies {
    annotationProcessor("org.springframework.boot:spring-boot-configuration-processor")
}
```
