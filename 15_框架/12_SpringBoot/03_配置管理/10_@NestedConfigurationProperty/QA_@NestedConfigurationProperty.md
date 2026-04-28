# @NestedConfigurationProperty QA

## Q1：@NestedConfigurationProperty 是必须的吗？

**A**：不是必须的，但推荐加。它的作用是让 IDE 提示更准确，不影响功能。

---

## Q2：不加这个注解会有什么后果？

**A**：功能上没问题，只是 IDE 的 YAML 提示可能不准确：

```yaml
# 不加注解
server:
  servlet: ...  # IDE 不知道这是嵌套对象

# 加注解后
server:
  servlet:
    contextPath: ...  # IDE 正确提示
```

---

## Q3：哪些情况需要加？

**A**：
- 配置类有嵌套 POJO
- 希望 IDE 提示层级化
- 使用了 spring-boot-configuration-processor

---

## Q4：可以用 Lombok 简化吗？

**A**：可以：

```java
@ConfigurationProperties(prefix = "server")
@Data
public class ServerProperties {

    private int port = 8080;

    @NestedConfigurationProperty
    private Servlet servlet = new Servlet();

    @Data
    public static class Servlet {
        private String contextPath = "/";
    }
}
```

---

## Q5：嵌套层级深的话，每层都要加吗？

**A**：建议每层嵌套对象都加上：

```java
@NestedConfigurationProperty  // 第一层
private A a;

@Data
public static class A {
    @NestedConfigurationProperty  // 第二层
    private B b;
}
```
