# 条件注解总结对比

## Q1：所有条件注解的执行顺序是什么？

**A**：按检查时机分组：

```
1. classpath 扫描阶段（最早）
   ├── @ConditionalOnClass
   ├── @ConditionalOnMissingClass
   └── @ConditionalOnResource

2. BeanDefinition 注册阶段
   └── @AutoConfigurationImportFilter

3. Bean 实例化阶段（最晚）
   ├── @ConditionalOnBean
   ├── @ConditionalOnMissingBean
   ├── @ConditionalOnProperty
   ├── @ConditionalOnWebApplication
   ├── @ConditionalOnNotWebApplication
   └── @ConditionalOnExpression
```

---

## Q2：如何组合使用多个条件注解？

**A**：多个条件注解是 AND 关系（全部满足才加载）：

```java
@ConditionalOnClass(DataSource.class)              // 有数据源依赖
@ConditionalOnProperty(name = "my.dao.enabled")  // 功能已启用
@ConditionalOnMissingBean(MyDao.class)            // 没有自定义实现
public class DefaultMyDaoAutoConfiguration { }
// 三个条件都满足才加载
```

---

## Q3：什么时候用 @ConditionalOnProperty 替代 @ConditionalOnExpression？

**A**：

| 场景 | 推荐 |
|------|------|
| 简单布尔开关 | `@ConditionalOnProperty(name = "xxx.enabled", havingValue = "true")` |
| 带默认值 | `@ConditionalOnProperty(name = "xxx.enabled", havingValue = "true", matchIfMissing = true)` |
| 数值比较 | `@ConditionalOnExpression("${xxx.timeout:3000} > 5000")` |
| 多属性组合 | `@ConditionalOnExpression("${xxx.a} and ${xxx.b}")` |

**原则**：能用 `@ConditionalOnProperty` 就不用 `@ConditionalOnExpression`。

---

## Q4：条件注解写在类上和方法上有什么区别？

**A**：

| 位置 | 影响范围 | 典型用途 |
|------|----------|----------|
| **类上** | 整个配置类 | 需要多个条件都满足才加载整个类 |
| **方法上** | 单个 @Bean 方法 | 只有该 Bean 需要特定条件 |

```java
// 类级别：整个类都不加载
@ConditionalOnClass(Missing.class)  // 整个类跳过
public class Config {
    @Bean public A a() { return new A(); }
    @Bean public B b() { return new B(); }  // 也不注册
}

// 方法级别：只有该 Bean 不注册
public class Config {
    @Bean public A a() { return new A(); }  // 正常注册
    @Bean @ConditionalOnClass(Missing.class) public B b() { return new B(); }  // 不注册
}
```

---

## Q5：如何快速选择合适的条件注解？

**A**：对照下表：

| 你想检查什么？ | 使用哪个注解？ |
|---------------|---------------|
| classpath 有某个类 | `@ConditionalOnClass` |
| classpath 没有某个类 | `@ConditionalOnMissingClass` |
| 容器中有某个 Bean | `@ConditionalOnBean` |
| 容器中没有某个 Bean | `@ConditionalOnMissingBean` |
| 配置属性满足条件 | `@ConditionalOnProperty` |
| 存在某个资源文件 | `@ConditionalOnResource` |
| 是 Web 环境 | `@ConditionalOnWebApplication` |
| 不是 Web 环境 | `@ConditionalOnNotWebApplication` |
| 复杂的条件表达式 | `@ConditionalOnExpression` |

