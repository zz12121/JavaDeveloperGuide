# @SpringBootApplication注解三合一 - QA

## Q1：@SpringBootApplication 和 @SpringBootConfiguration 有什么区别？

**A**：
- `@SpringBootConfiguration` 是 `@Configuration` 的别名，本质相同
- `@SpringBootApplication` 是组合注解，包含 `@SpringBootConfiguration`
- 新版本中 `@SpringBootApplication` 已直接包含 `@Configuration`，`@SpringBootConfiguration` 使用较少

**建议**：直接使用 `@SpringBootApplication` 即可。

---

## Q2：@ComponentScan 会扫描到第三方 jar 包中的组件吗？

**A**：默认不会。

**原因**：`@ComponentScan` 默认只扫描主类所在包及其子包。

**解决方案**：
```java
// 方案1：添加扫描路径
@SpringBootApplication
@ComponentScan(basePackages = {"com.example.demo", "com.example.thirdparty"})

// 方案2：使用 @Import 导入
@Import(ThirdPartyConfiguration.class)
@SpringBootApplication
public class Application {}
```

---

## Q3：如何禁用特定的自动配置？

**A**：有三种方式：

```java
// 方式1：exclude 参数（最常用）
@SpringBootApplication(exclude = {DataSourceAutoConfiguration.class})

// 方式2：excludeName 参数
@SpringBootApplication(excludeName = "org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration")

// 方式3：配置文件
spring.autoconfigure.exclude=org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```

---

## Q4：@EnableAutoConfiguration 的执行时机是什么？

**A**：
- 在应用上下文刷新时执行
- 通过 `AutoConfigurationImportSelector` 实现
- 读取 `META-INF/spring.factories` 中的自动配置类
- 按 `@Conditional*` 条件逐一筛选，符合条件的才生效

---

## Q5：可以把启动类放在 jar 包中吗？

**A**：不推荐。

**原因**：
- `@ComponentScan` 扫描启动类所在包
- 如果放在第三方 jar 中，该 jar 的组件可能扫描不到
- 最佳实践：启动类放在最外层包，组件放在子包

**标准项目结构**：
```
com.example.demo
├── Application.java          ← 启动类，最外层
├── controller/
├── service/
├── dao/
└── config/
```
