# Spring Boot 可执行 JAR 原理

## Q1：Spring Boot JAR 和普通 JAR 有什么区别？

**A**：

| 对比 | 普通 JAR | Spring Boot 可执行 JAR |
|------|---------|----------------------|
| **结构** | 扁平，所有类在同一层级 | BOOT-INF 嵌套结构 |
| **依赖** | 需要放到 lib/ 并通过 classpath 引用 | 依赖打包在 BOOT-INF/lib/ |
| **Main-Class** | 应用自己的 main 类 | JarLauncher |
| **执行方式** | `java -cp xxx.jar` | `java -jar xxx.jar` |
| **依赖加载** | 系统 ClassLoader | 自定义 LaunchedURLClassLoader |

---

## Q2：Main-Class 和 Start-Class 的区别？

**A**：

| 属性 | 说明 |
|------|------|
| **Main-Class** | JVM 实际加载的类，必须有 main 方法 |
| **Start-Class** | 应用的入口类，包含真正的业务 main 方法 |

**执行流程**：
```
JarLauncher.main()  ← Main-Class（JVM 加载）
    ↓
加载 Start-Class
    ↓
Application.main()  ← Start-Class（真正的业务入口）
```

---

## Q3：Spring Boot 如何加载嵌套 JAR 中的类？

**A**：通过 `LaunchedURLClassLoader` 和自定义文件系统。

**加载过程**：
```java
// JarLauncher 源码简化
public class JarLauncher extends ExecutableArchiveLauncher {
    
    @Override
    protected ClassLoader createClassLoader() throws Exception {
        // 创建 URLClassLoader
        URLs urls = new URLs();
        
        // 添加 BOOT-INF/classes
        urls.add("BOOT-INF/classes/");
        
        // 添加 BOOT-INF/lib/*.jar
        for (NestedJarFile jar : nestedJars) {
            urls.add(jar.getURL());
        }
        
        return new LaunchedURLClassLoader(urls);
    }
}
```

**关键点**：
- 使用 `java.util.jar.JarFile` API
- 通过 `JarIndex` 索引快速定位资源
- 利用 `URLClassLoader` 加载嵌套 JAR

---

## Q4：如何查看可执行 JAR 的内容？

**A**：

**方式一：jar 命令**
```bash
jar tf myapp.jar
```

**方式二：unzip**
```bash
unzip -l myapp.jar
```

**方式三：Maven**
```bash
mvn dependency:tree
mvn spring-boot:repackage  # 重新打包
```

---

## Q5：如何指定应用的 Main-Class？

**A**：

**方式一：pom.xml**
```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <mainClass>com.example.Application</mainClass>
    </configuration>
</plugin>
```

**方式二：命令行**
```bash
mvn spring-boot:repackage -DmainClass=com.example.Application
```

**方式三：application.yml**
```yaml
spring:
  main:
    main-class: com.example.Application  # 注意：这只影响 Banner 显示
```

---

## Q6：可执行 JAR 能解压后运行吗？

**A**：**不能直接运行**，但可以：

**方式一：使用 classpath**
```bash
java -cp "BOOT-INF/classes:BOOT-INF/lib/*" com.example.Application
```

**方式二：修改 MANIFEST.MF**
```properties
Main-Class: com.example.Application
```

---

## Q7：Spring Boot 2.3+ 的分层 JAR 是什么？

**A**：分层 JAR 可以将应用分层提取，提高 Docker 镜像构建效率。

**分层结构**：
```
app.jar
├── BOOT-INF/
│   ├── classes/
│   ├── classpath.idx     ← 依赖索引
│   └── layers.idx         ← 分层索引
│   └── lib/
│       ├── spring-boot-3.2.0.jar
│       └── ...
```

**Docker 多阶段构建**：
```dockerfile
# 第一阶段：提取依赖层
FROM build AS builder
COPY target/app.jar /app.jar
RUN java -Djarmode=layertools extract app.jar

# 第二阶段：只复制依赖层
FROM runtime
COPY --from=builder /app/dependencies/ ./
COPY --from=builder /app/spring-boot-loader/ ./
COPY --from=builder /app/snapshot-dependencies/ ./
COPY --from=builder /app/application/ ./
ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

---

## Q8：如何排除某些依赖不打包？

**A**：

```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <excludeGroupIds>
            <excludeGroupId>org.projectlombok</excludeGroupId>
        </excludeGroupIds>
        <excludes>
            <exclude>
                <groupId>javax.servlet</groupId>
                <artifactId>servlet-api</artifactId>
            </exclude>
        </excludes>
    </configuration>
</plugin>
```
