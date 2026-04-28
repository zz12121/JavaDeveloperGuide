# AutoConfigurationImportSelector

## Q1：AutoConfigurationImportSelector 在哪个阶段执行？

**A**：`DeferredImportSelector` 在所有普通 `@Configuration` 类处理完成后执行，具体阶段：

```
容器启动
    │
    ▼
ConfigurationClassPostProcessor 处理所有 @Configuration
    │
    ├── 扫描 @ComponentScan 下的配置类
    ├── 处理 @Import 引入的普通 ImportSelector
    └── 处理 @PropertySource、@Bean 等
    │
    ▼
处理 @Import(AutoConfigurationImportSelector.class)
    └── 调用 selectImports() 获取自动配置类
    │
    ▼
注册自动配置类为 BeanDefinition
```

关键：它在 `@ComponentScan` 之后执行，意味着用户代码中的 Bean 已经注册完成，`@ConditionalOnMissingBean` 可以正确检测。

---

## Q2：它和普通的 ImportSelector 有什么区别？

**A**：继承链：`ImportSelector` → `DeferredImportSelector`

| 特性 | ImportSelector | DeferredImportSelector |
|------|----------------|----------------------|
| 执行时机 | 配置类解析阶段立即执行 | 所有配置类解析完成后执行 |
| 与 @ComponentScan 关系 | 可能早于用户 Bean | **一定晚于**用户 Bean |
| 应用场景 | 普通功能扩展 | 自动配置（必须后执行） |
| 代码位置 | 直接实现接口 | 继承 `AutoConfigurationImportSelector` |

**为什么自动配置必须用 DeferredImportSelector？**

因为 `@ConditionalOnMissingBean` 需要在用户手动配置的 Bean 之后才判断，否则用户配置会被覆盖。

---

## Q3：AutoConfigurationImportSelector 返回的配置类一定生效吗？

**A**：不一定。返回的只是候选配置类列表，最终是否生效还取决于：

1. **过滤** — `AutoConfigurationImportFilter` 会过滤掉不满足基本条件的配置
2. **条件判断** — 每个配置类的 `@Conditional*` 注解会逐一验证
3. **去重** — 同一配置类不会重复加载

---

## Q4：如何自定义 AutoConfigurationImportSelector？

**A**：通常不直接继承此类，而是通过 `spring.factories` 或 `AutoConfiguration.imports` 注册自定义配置类。但如果确实需要扩展：

```java
public class MyAutoConfigurationImportSelector
        extends AutoConfigurationImportSelector {

    @Override
    protected List<String> getCandidateConfigurations(
            AnnotationMetadata metadata, AnnotationAttributes attributes) {
        List<String> configs = super.getCandidateConfigurations(metadata, attributes);
        // 添加自定义配置类
        configs.add("com.example.MyAutoConfiguration");
        return configs;
    }
}
```

然后在 `spring.factories` 中注册：
```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  com.example.config.MyAutoConfigurationImportSelector
```

**注意**：这种方式会改变全局自动配置行为，一般不推荐。
