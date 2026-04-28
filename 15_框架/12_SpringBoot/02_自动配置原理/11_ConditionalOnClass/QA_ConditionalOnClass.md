# @ConditionalOnClass 条件判断

## Q1：@ConditionalOnClass 的 name 和 value 属性有什么区别？

**A**：

| 属性 | 类型 | 特点 |
|------|------|------|
| `value` | `Class<?>[]` | 编译时检查，类不存在则编译失败 |
| `name` | `String[]` | 运行时检查，更安全 |

```java
// Class 引用：必须确保类存在，否则编译失败
@ConditionalOnClass(DataSource.class)  // spring-boot-starter-data-jdbc 引入后才有

// String 类名：即使类不存在也能编译通过
@ConditionalOnClass(name = "com.example.OptionalClass")
```

**最佳实践**：
- 确定在 classpath 中的依赖：用 `value`
- 可选的依赖：用 `name`

---

## Q2：@ConditionalOnClass 检查的是类的什么状态？

**A**：只检查类是否存在于 classpath，**不检查是否能实例化**：

```java
// 即使 MyClass 有静态初始化块错误，也会通过条件判断
@ConditionalOnClass(MyClass.class)  // classpath 有 .class 文件即可
```

真正的实例化发生在条件通过、Bean 注册之后，此时才会触发类加载错误。

---

## Q3：多个 @ConditionalOnClass 是什么关系？

**A**：**AND（且）关系**——所有类都存在才通过：

```java
@ConditionalOnClass({DataSource.class, JdbcTemplate.class})
// 必须同时满足：
//   - classpath 有 DataSource
//   - classpath 有 JdbcTemplate
// 两个都满足才加载
```

---

## Q4：如何在自定义 Starter 中使用 @ConditionalOnClass？

**A**：

```java
// pom.xml 中 dependencyManagement 标记为 optional
<dependency>
    <groupId>com.example</groupId>
    <artifactId>my-lib</artifactId>
    <optional>true</optional>  <!-- 可选依赖 -->
</dependency>
```

```java
// 自动配置类
@AutoConfiguration
// 只有 classpath 有 MyLib 类时才加载
@ConditionalOnClass(MyLib.class)
public class MyLibAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public MyLib myLib() {
        return new MyLib();
    }
}
```

---

## Q5：条件不满足会抛异常吗？

**A**：**不会**，是静默跳过：

```java
// UserAutoConfig 在 classpath 没有 MyService 类时不加载
// 但不会抛出任何异常
@ConditionalOnClass(MyService.class)
public class UserAutoConfig { }

// 如果开发者期望此配置生效而忘记引入依赖
// 排查困难：启动正常但功能缺失
```

**建议**：开启 `debug: true` 查看自动配置报告，或查看启动日志中的条件判断结果。

