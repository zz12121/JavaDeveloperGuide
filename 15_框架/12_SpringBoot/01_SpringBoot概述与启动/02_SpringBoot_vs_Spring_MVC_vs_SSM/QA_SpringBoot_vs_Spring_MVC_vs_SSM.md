# Spring Boot vs Spring MVC vs SSM

## Q1：SSM 和 Spring Boot 的主要区别是什么？

**A**：

| 区别 | SSM | Spring Boot |
|------|-----|-------------|
| **配置方式** | XML 配置文件 | 注解 + 自动配置 |
| **依赖管理** | 手动添加 jar，手动指定版本 | starter 依赖 + parent 版本管理 |
| **容器** | 需要部署到外部 Tomcat | 内嵌 Tomcat/Jetty/Undertow |
| **打包** | War 包 | Jar 包（默认） |
| **启动** | 把 War 部署到 Tomcat 启动 | java -jar 直接运行 |
| **配置量** | 大量 XML | 少量或零配置 |

**SSM 配置示例（大量 XML）**：
```xml
<!-- applicationContext.xml -->
<bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource">
    <property name="url" value="jdbc:mysql://localhost:3306/test"/>
    <property name="username" value="root"/>
    <property name="password" value="123456"/>
</bean>

<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
    <property name="dataSource" ref="dataSource"/>
    <property name="mapperLocations" value="classpath:mapper/*.xml"/>
</bean>

<bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
    <property name="basePackage" value="com.example.mapper"/>
</bean>
```

**Spring Boot 配置（只需 application.yml）**：
```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/test
    username: root
    password: 123456
  jpa:
    hibernate:
      ddl-auto: update
```

---

## Q2：Spring MVC 和 Spring Boot 是什么关系？

**A**：**Spring Boot 是 Spring MVC 的增强版，不是替代关系**。

```
关系图：
┌─────────────────────────────────────────┐
│              Spring Framework           │
│  ┌─────────────────────────────────┐   │
│  │         Spring MVC              │   │
│  │  (Web 框架：控制器/视图/解析器)    │   │
│  └─────────────────────────────────┘   │
│                 ↑                      │
│          ┌──────┴──────┐               │
│          │ Spring Boot │               │
│          │  + 自动配置  │               │
│          │  + 内嵌容器  │               │
│          │  + Starter  │               │
│          └─────────────┘               │
└─────────────────────────────────────────┘
```

**Spring Boot 包含的内容**：
- Spring Framework（IoC、AOP）
- Spring MVC（Web 框架）
- 自动配置（WebMvcAutoConfiguration 等）
- 内嵌 Servlet 容器
- starter 依赖管理

---

## Q3：为什么说 Spring Boot 可以"零配置"开发？

**A**：Spring Boot 通过以下三板斧实现零配置：

**1. Starter 依赖自动引入**
```xml
<!-- 引入 web starter，自动引入所有需要的 jar -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

**2. 自动配置类生效**
```
spring-boot-autoconfigure.jar 中包含：
- WebMvcAutoConfiguration（Spring MVC 自动配置）
- DataSourceAutoConfiguration（数据源自动配置）
- JdbcTemplateAutoConfiguration（JDBC 自动配置）
- ...
```

**3. 提供合理的默认值**
```
默认端口：8080
默认上下文路径：/
默认日志：SLF4J + Logback
默认 JSON：Jackson
```

**开发者只需关注业务代码**，框架层面的配置 Spring Boot 已经做好。

---

## Q4：什么场景下不推荐使用 Spring Boot？

**A**：虽然 Spring Boot 是主流选择，但在以下场景可能不适用：

| 场景 | 原因 | 替代方案 |
|------|------|----------|
| **遗留系统** | 需要继续兼容原有 XML 配置 | 保持 SSM |
| **特定容器** | 必须部署到 WebSphere/WebLogic | 打包成 War |
| **极简需求** | 只是工具类，不需要 Web | 纯 Java 应用 |
| **资源受限** | 嵌入式设备，内存极小 | Spring Framework 精简版 |

---

## Q5：Spring Boot 相比 SSM 的优势总结

**A**：

| 优势 | 说明 |
|------|------|
| **快速构建** | 几秒钟创建一个可运行项目 |
| **自动配置** | 减少 80% 的配置工作 |
| **内嵌容器** | 无需安装配置 Tomcat |
| **独立运行** | java -jar 一键启动 |
| **健康检查** | Actuator 提供监控端点 |
| **易于部署** | 支持 Docker、K8s 原生 |
| **生态丰富** | 大量官方和第三方 Starter |
| **版本统一** | parent 解决依赖冲突 |
