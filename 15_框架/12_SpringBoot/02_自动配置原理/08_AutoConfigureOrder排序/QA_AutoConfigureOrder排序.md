# @AutoConfigureOrder 排序

## Q1：@AutoConfigureOrder 和 @AutoConfigureBefore/After 哪个优先级高？

**A**：`@AutoConfigureOrder` 优先级更高：

```java
// 虽然声明了 @AutoConfigureAfter，但 @AutoConfigureOrder 显式指定了顺序
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE)
@AutoConfigureAfter(DataSourceAutoConfiguration.class)  // 被覆盖
public class MyAutoConfiguration { }
// 结果：会优先于 DataSourceAutoConfiguration 加载
```

**建议**：
- 简单相对顺序：用 `@AutoConfigureBefore/After`
- 精确控制：用 `@AutoConfigureOrder`

---

## Q2：如何用 @AutoConfigureOrder 实现"最后加载"？

**A**：

```java
@AutoConfiguration
@AutoConfigureOrder(Ordered.LOWEST_PRECEDENCE - 1)  // 最后加载
public class OverrideAutoConfiguration {
    // 可以覆盖之前所有自动配置的 Bean
    @Bean
    @ConditionalOnMissingBean
    public MyService myService() {
        return new CustomMyService();  // 优先级最高的实现
    }
}
```

**注意**：`LOWEST_PRECEDENCE` 本身是 `Integer.MAX_VALUE`，`LOWEST_PRECEDENCE - 1` 反而更高。

---

## Q3：多个配置使用相同的 @AutoConfigureOrder 值会怎样？

**A**：按类名字母顺序排序：

```java
// AAutoConfiguration 和 BAutoConfiguration 都是 LOWEST_PRECEDENCE
@AutoConfigureOrder(Ordered.LOWEST_PRECEDENCE)  // 相同顺序
public class AAutoConfiguration { }  // A 在前

@AutoConfigureOrder(Ordered.LOWEST_PRECEDENCE)  // 相同顺序
public class BAutoConfiguration { }  // B 在后
```

---

## Q4：自定义 Starter 如何确保在官方配置之后加载？

**A**：

```java
@AutoConfiguration
@AutoConfigureOrder(Ordered.LOWEST_PRECEDENCE - 10)
public class MyStarterAutoConfiguration {
    // 此时官方所有自动配置都已完成
}
```

或者：

```java
@AutoConfiguration
@AutoConfigureAfter({
    DataSourceAutoConfiguration.class,
    JdbcTemplateAutoConfiguration.class,
    TransactionAutoConfiguration.class
})
public class MyStarterAutoConfiguration { }
```

