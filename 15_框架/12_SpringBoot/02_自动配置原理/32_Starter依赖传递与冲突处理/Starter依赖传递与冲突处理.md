# Starter 依赖传递与冲突处理

## 先说结论

- **传递依赖**：引入 Starter A 会自动引入 A 依赖的所有包
- **版本冲突**：Spring Boot 提供 `spring-boot-dependencies` 统一管理版本
- **冲突处理**：排除（exclude）+ 版本锁定（dependencyManagement）+ 软替换

## 深度解析

### 传递依赖原理

```
用户引入：spring-boot-starter-web
  │
  ├── spring-boot-starter (核心)
  │     ├── spring-core
  │     ├── spring-context
  │     └── spring-boot
  │
  ├── spring-boot-starter-tomcat
  │     └── tomcat-embed-core
  │
  ├── jackson-databind
  │     ├── jackson-core
  │     └── jackson-annotations
  │
  └── spring-web
        ├── spring-beans
        └── spring-webmvc
```

### spring-boot-dependencies 核心机制

```xml
<!-- spring-boot-dependencies 统一管理版本 -->
<dependencyManagement>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-core</artifactId>
        <version>${spring-framework.version}</version>
    </dependency>
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
        <version>${jackson.version}</version>
    </dependency>
</dependencyManagement>
```

### 常见冲突场景

| 冲突 | 表现 | 解决 |
|------|------|------|
| MyBatis 3.5 vs 3.4 | NoSuchMethodError | 版本锁定 |
| Jackson 版本 | JSON 解析异常 | 排除 + 统一版本 |
| 日志框架 SLF4J | 多个绑定 | exclude + 统一 |
| HikariCP 连接池 | 多数据源冲突 | exclude |

## 易错点/踩坑

- ❌ **引入多个日志框架**：同时引入 logback + log4j
- ❌ **版本冲突不报错**：只在运行时爆炸
- ❌ **忽略传递依赖**：引入 A 实际引入了一堆不需要的

## 代码示例

### 排除传递依赖

```xml
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>2.3.0</version>
    <exclusions>
        <!-- 排除自带的 HikariCP，使用 Druid -->
        <exclusion>
            <groupId>com.zaxxer</groupId>
            <artifactId>HikariCP</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

### 版本锁定

```xml
<properties>
    <springfox.version>2.10.0</springfox.version>
</properties>

<dependencyManagement>
    <dependency>
        <groupId>io.springfox</groupId>
        <artifactId>springfox-boot-starter</artifactId>
        <version>${springfox.version}</version>
    </dependency>
</dependencyManagement>
```

## 关联知识点

- Spring Boot Starter 命名规范
- 自定义 Starter 开发
- spring-boot-maven-plugin
