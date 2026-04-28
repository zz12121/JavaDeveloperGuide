# Spring Boot 核心优势

## Q1：Spring Boot 相比传统 Spring 的三大核心优势是什么？

**A**：

| 优势 | 说明 | 带来的价值 |
|------|------|------------|
| **快速构建** | 通过 Initializr 或 IDE 向导，勾选依赖即可生成完整项目 | 搭建项目从 1 天缩短到 5 分钟 |
| **自动配置** | 提供 200+ 自动配置类，根据类路径自动生效 | 减少 80% 的配置工作 |
| **内嵌容器** | Tomcat/Jetty/Undertow 随应用一起打包 | 告别 Tomcat 安装配置，java -jar 运行 |

---

## Q2：Spring Boot 如何实现"开箱即用"？

**A**：通过 **starter 依赖** + **自动配置** 实现：

**1. starter 依赖**
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <!-- 自动引入：Spring MVC + Tomcat + Jackson + Web -->
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
    <!-- 自动引入：Spring Data JPA + Hibernate + JPA -->
</dependency>
```

**2. 自动配置生效条件**
```
类路径检测 → 发现 spring-boot-starter-data-jpa
    ↓
HibernateJpaAutoConfiguration 生效
    ↓
自动配置 EntityManagerFactory、TransactionManager 等
```

**3. 默认配置值**
```yaml
# 只需覆盖需要改的配置
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/test
    username: root
    password: 123456
# 其他（连接池大小、超时等）全部使用默认值
```

---

## Q3：内嵌容器有什么优势？

**A**：

| 传统方式（部署到 Tomcat） | Spring Boot（内嵌容器） |
|--------------------------|-------------------------|
| 需要安装 Tomcat | 无需安装，随应用打包 |
| 需要配置 Tomcat | 零配置或少量配置 |
| 每次改代码需要重新部署到 Tomcat | 直接运行 main 方法 |
| 多应用需要多个 Tomcat 实例 | 每个应用独立运行 |
| 依赖 Tomcat 版本 | 可自由选择容器版本 |

**启动方式对比**：
```bash
# 传统：先启动 Tomcat，再部署 war
$ catalina.sh start
$ cp myapp.war $CATALINA_HOME/webapps/

# Spring Boot：直接运行
$ java -jar myapp.jar
```

---

## Q4：Spring Boot starter 的命名规范？

**A**：

| 类型 | 命名格式 | 说明 | 示例 |
|------|----------|------|------|
| **官方 starter** | spring-boot-starter-{功能} | Spring 官方维护 | spring-boot-starter-web |
| **自动移除 starter** | spring-boot-starter-{功能} | 提供自动配置 | spring-boot-starter-tomcat |
| **第三方 starter** | {功能}-spring-boot-starter | 第三方维护 | mybatis-spring-boot-starter |

**官方 vs 第三方**：
```xml
<!-- 官方 starter -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<!-- 第三方 starter -->
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
</dependency>
```

---

## Q5：Spring Boot 的自动配置能完全替代手动配置吗？

**A**：**不能完全替代**，但能覆盖 90% 的常见场景。

**自动配置擅长**：
- 标准配置（大多数项目都是这样配置的）
- 常见场景（单数据源、单数据源缓存等）

**仍需手动配置**：
- 多数据源配置
- 复杂的安全策略
- 自定义拦截器/过滤器
- 非标准的第三方组件集成

**混合使用示例**：
```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}

// 自定义配置类 - 在自动配置基础上添加
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {
    
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new LoginInterceptor())
                .addPathPatterns("/api/**");
    }
}
```

---

## Q6：Spring Boot 如何切换嵌入式容器？

**A**：通过排除默认 + 引入新的依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <!-- 排除默认的 Tomcat -->
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>

<!-- 引入 Jetty -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jetty</artifactId>
</dependency>
```

**容器选型建议**：
- **Tomcat**：默认选择，生态完善
- **Jetty**：嵌入式设备，内存受限场景
- **Undertow**：高并发高性能需求，不支持 JSP
