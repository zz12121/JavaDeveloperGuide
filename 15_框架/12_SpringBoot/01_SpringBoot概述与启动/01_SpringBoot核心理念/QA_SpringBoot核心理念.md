# Spring Boot 核心理念

## Q1：什么是约定大于配置？

**A**：约定大于配置（Convention Over Configuration）是一种软件设计原则，框架为常见场景预定义默认值，开发者只在偏离默认行为时才进行显式配置。

**Spring Boot 中的体现**：
- 项目目录结构约定（src/main/java、src/main/resources）
- 配置文件位置和命名约定（application.properties/yml）
- 自动配置类路径约定（META-INF/spring.factories）
- 静态资源目录约定（/static、/public 等）

---

## Q2：Spring Boot 如何实现"零配置"？

**A**：Spring Boot 通过以下机制实现零配置：

1. **自动配置（Auto Configuration）**
   - `spring-boot-autoconfigure` jar 包包含大量自动配置类
   - 通过 `@Conditional*` 条件注解判断是否生效

2. **starter 依赖**
   - 引入 starter 后自动引入相关依赖
   - parent pom 统一管理版本号

3. **默认配置值**
   - 所有配置项都有默认值
   - 只需在 application.properties 中覆盖需要改的部分

```java
// 引入 web starter 后，自动配置以下内容：
// - EmbeddedServletContainerAutoConfiguration（内嵌容器）
// - HttpMessageConvertersAutoConfiguration（JSON转换）
// - JacksonAutoConfiguration（Jackson支持）
// - WebMvcAutoConfiguration（Spring MVC）
```

---

## Q3：约定大于配置有什么好处？

**A**：

| 好处 | 说明 |
|------|------|
| **减少样板代码** | 不需要写大量 XML 或 Java 配置 |
| **提高开发效率** | 开箱即用，快速启动项目 |
| **降低学习成本** | 只需学习偏离默认的部分 |
| **统一团队风格** | 大家遵循相同的约定 |
| **减少错误** | 默认配置经过充分测试 |

---

## Q4：约定大于配置有什么局限？

**A**：

1. **特殊场景不适用**
   - 业务复杂时可能需要大量自定义配置
   - 某些非标准需求无法通过默认配置实现

2. **隐藏复杂性**
   - 默认配置屏蔽了底层细节
   - 出问题时排查难度增加

3. **升级风险**
   - 自动配置类由 Spring Boot 提供
   - 版本升级可能导致行为变化

**解决方案**：查看自动配置报告（debug 模式）

```properties
# 开启自动配置报告
debug=true
```

---

## Q5：如何覆盖默认配置？

**A**：有多种方式覆盖 Spring Boot 的默认配置：

**方式一：application.properties/yml**

```properties
server.port=9090
server.servlet.context-path=/demo
spring.datasource.url=jdbc:mysql://localhost:3306/test
```

**方式二：代码方式**

```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication app = new SpringApplication(Application.class);
        app.setBannerMode(Banner.Mode.OFF);
        app.setPort(9090);
        app.run(args);
    }
}
```

**方式三：命令行参数**

```bash
java -jar app.jar --server.port=9090
```

**优先级**：命令行参数 > application.properties > 默认值
