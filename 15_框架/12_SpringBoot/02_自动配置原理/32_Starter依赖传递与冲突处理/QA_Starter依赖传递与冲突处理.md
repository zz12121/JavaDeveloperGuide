# Starter 依赖传递与冲突处理

## Q1：Spring Boot 如何统一管理依赖版本？

**A**：通过 `spring-boot-dependencies` BOM（Bill of Materials）。

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>3.2.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

**效果**：引入后，所有 Starter 的依赖版本都由 Spring Boot 统一指定，无需手动管理。

---

## Q2：如何排除传递依赖？

**A**：使用 `<exclusions>` 标签。

```xml
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <exclusions>
        <exclusion>
            <groupId>com.zaxxer</groupId>
            <artifactId>HikariCP</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

**Gradle 方式**：
```groovy
implementation("org.mybatis.spring.boot:mybatis-spring-boot-starter") {
    exclude group: "com.zaxxer", module: "HikariCP"
}
```

---

## Q3：常见的传递依赖冲突有哪些？

**A**：

| 冲突类型 | 示例 | 症状 |
|---------|------|------|
| 日志框架 | logback + log4j | NoSuchMethodError |
| JSON库 | Jackson + Gson | 类型转换异常 |
| Servlet版本 | Servlet 3 vs 4 | API不存在 |
| 连接池 | HikariCP + Druid | 多数据源 |

---

## Q4：如何调试依赖冲突？

**A**：使用 Maven/Gradle 插件查看依赖树。

**Maven**：
```bash
mvn dependency:tree -Dincludes=*:jackson*
```

**IDEA**：右侧 Maven 面板 → Dependencies → 搜索

**Gradle**：
```bash
gradle dependencies --configuration runtimeClasspath
```

---

## Q5：为什么建议自定义 Starter 不要带版本号？

**A**：保持与 Spring Boot 版本一致。

```xml
<!-- ✅ 推荐：不写版本 -->
<dependency>
    <groupId>com.mycompany</groupId>
    <artifactId>mycompany-spring-boot-starter</artifactId>
</dependency>

<!-- ❌ 不推荐：写死版本 -->
<dependency>
    <groupId>com.mycompany</groupId>
    <artifactId>mycompany-spring-boot-starter</artifactId>
    <version>1.0.0</version>
</dependency>
```

让 `spring-boot-dependencies` 统一管理，避免版本碎片化。
