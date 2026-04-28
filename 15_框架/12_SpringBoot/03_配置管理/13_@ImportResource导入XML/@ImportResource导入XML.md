# @ImportResource 导入 XML

## 先说结论

`@ImportResource` 用于导入 Spring XML 配置文件，适合需要复用遗留 XML 配置或引入不支持注解配置的第三方库的场景。

## 深度解析

### 基本用法

```java
@Configuration
@ImportResource("classpath:beans.xml")
public class AppConfig {
    // 导入 classpath:beans.xml 中的 Bean 定义
}
```

### XML 配置文件示例

```xml
<!-- src/main/resources/beans.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
           http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="dataSource" class="com.zaxxer.hikari.HikariDataSource">
        <property name="jdbcUrl" value="jdbc:mysql://localhost:3306/test"/>
        <property name="username" value="root"/>
        <property name="password" value="123456"/>
    </bean>
</beans>
```

### Spring Boot 中的等价配置

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/test
    username: root
    password: 123456
```

### 导入多个 XML

```java
@ImportResource({"classpath:beans.xml", "classpath:daos.xml"})
```

### 混合注解和 XML

```java
@Configuration
@ImportResource("classpath:legacy-beans.xml")
public class AppConfig {

    @Bean
    public MyService myService() {
        return new MyService();
    }
}
```

## 易错点/踩坑

- ❌ XML 中的占位符 `${}` 解析 — 需要 PropertySourcesPlaceholderConfigurer
- ❌ 混用导致配置混乱 — 尽量统一用 Java/YAML 配置
- ❌ XML 和注解配置冲突 — Bean 名称相同会被覆盖

## 关联知识点

- [[11_@PropertySource加载外部配置]] — 加载外部配置
- [[14_SpringBoot配置优先级]] — 配置加载顺序
