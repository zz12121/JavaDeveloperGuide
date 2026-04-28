# @ConditionalOnMissingClass 条件判断

## Q1：@ConditionalOnMissingClass 和 @ConditionalOnMissingBean 有什么区别？

**A**：

| 对比项 | @ConditionalOnMissingClass | @ConditionalOnMissingBean |
|--------|---------------------------|---------------------------|
| 检查内容 | classpath 中是否存在某类 | 容器中是否已注册某 Bean |
| 检查时机 | 配置类加载前 | Bean 注册时 |
| 应用场景 | jar 依赖判断 | Bean 覆盖判断 |
| 属性类型 | 只能是 String（类名字符串） | 可以是 Class 或 String |

```java
// 检查 classpath
@ConditionalOnMissingClass("com.alibaba.fastjson.JSON")
// 检查容器
@ConditionalOnMissingBean(ObjectMapper.class)
```

---

## Q2：为什么 @ConditionalOnMissingClass 只有 String 属性？

**A**：因为目的是"检查类不存在"，如果用 Class 引用，类不存在时根本无法编译：

```java
// ❌ 编译错误：MyOptionalClass 不存在
@ConditionalOnMissingClass(MyOptionalClass.class)

// ✅ 正确：字符串方式
@ConditionalOnMissingClass("com.example.MyOptionalClass")
```

---

## Q3：什么场景下使用 @ConditionalOnMissingClass？

**A**：典型场景是"提供替代方案"：

```java
// FastJson 存在时不加载 Jackson
@ConditionalOnMissingClass("com.alibaba.fastjson.JSON")
@Configuration
public class JacksonAutoConfiguration {
    // 提供 Jackson 作为默认 JSON 处理
}
```

另一个例子：Servlet 容器检测：
```java
// 只有没有 Jersey 时，才加载默认 JAX-RS 支持
@ConditionalOnMissingClass("org.glassfish.jersey.servlet.ServletContainer")
public class SpringMvcAutoConfiguration { }
```

---

## Q4：多个 @ConditionalOnMissingClass 如何组合？

**A**：与 `@ConditionalOnClass` 一样，是 AND 关系：

```java
@ConditionalOnMissingClass({
    "com.alibaba.fastjson.JSON",
    "com.google.gson.Gson"
})
// classpath 中既没有 FastJson 也没有 Gson 时才加载
public class JacksonAutoConfiguration { }
```

---

## Q5：和 @ConditionalOnClass 一起用是什么效果？

**A**：

```java
@ConditionalOnClass(ObjectMapper.class)                    // 一定有 ObjectMapper
@ConditionalOnMissingClass("com.alibaba.fastjson.JSON")    // 没有 FastJson
// 翻译：有 ObjectMapper 但没有 FastJson 时才加载
```

常用于：根据 classpath 情况选择不同的自动配置。

