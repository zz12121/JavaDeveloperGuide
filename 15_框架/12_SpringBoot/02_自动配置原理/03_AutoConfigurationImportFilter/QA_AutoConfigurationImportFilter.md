# AutoConfigurationImportFilter

## Q1：AutoConfigurationImportFilter 和 @Conditional* 注解有什么区别？

**A**：两者都是自动配置的"门卫"，但职责不同：

| 对比项 | AutoConfigurationImportFilter | @Conditional* 条件注解 |
|--------|------------------------------|----------------------|
| 执行时机 | **配置类加载之前**，快速预判 | 配置类加载过程中，逐个注解验证 |
| 作用范围 | 所有自动配置类全局过滤 | 只作用于当前配置类 |
| 灵活性 | 固定逻辑，不能自定义 | 可灵活组合多种条件 |
| 典型用途 | 排除 classpath 缺失的自动配置 | 精细化条件判断（属性、Bean 等） |
| 可扩展性 | 需实现接口并注册 | 直接注解即可 |

**关系**：Filter 先执行，剔除明显不满足的配置；Conditional 随后处理剩余配置类。

---

## Q2：Spring Boot 内置了哪几个过滤器？

**A**：三个：

| 过滤器 | 检查内容 |
|--------|----------|
| `OnClassCondition` | classpath 中是否存在指定类 |
| `OnBeanCondition` | 容器中是否存在/不存在指定 Bean |
| `OnWebApplicationCondition` | 当前应用是否是 Web 应用 |

可以通过 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 中的 `org.springframework.boot.autoconfigure.AutoConfigurationImportFilter` 配置来覆盖或添加自定义过滤器。

---

## Q3：可以自定义过滤器吗？

**A**：可以，但一般不推荐：

```java
public class MyAutoConfigurationFilter
        implements AutoConfigurationImportFilter {

    @Override
    public boolean[] match(String autoConfigurationClass,
                          AutoConfigurationMetadata metadata) {
        boolean match = autoConfigurationClass.startsWith("com.mycompany");
        return new boolean[] { match };
    }
}
```

注册方式：在 `AutoConfiguration.imports` 中添加：
```
org.springframework.boot.autoconfigure.AutoConfigurationImportFilter=\
com.mycompany.autoconfigure.MyAutoConfigurationFilter
```

**注意**：过度使用自定义过滤器会增加启动时间，不推荐。

---

## Q4：过滤器执行失败会怎样？

**A**：过滤器返回 `false` 的配置类会被直接排除，不会执行后续的 `@Conditional*` 条件判断，也不会注册任何 Bean。如果误判导致正常配置被排除，启动不会报错（静默跳过），需要在 `debug: true` 日志中查看。

