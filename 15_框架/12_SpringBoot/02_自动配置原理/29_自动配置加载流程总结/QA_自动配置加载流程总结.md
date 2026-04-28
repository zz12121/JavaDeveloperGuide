# 自动配置加载流程总结

## Q1：自动配置在 Spring 容器启动的哪个阶段执行？

**A**：在 `refresh()` 的 `ConfigurationClassPostProcessor` 阶段：

```java
AbstractApplicationContext.refresh()
    │
    ▼
invokeBeanFactoryPostProcessors()
    │
    ▼
ConfigurationClassPostProcessor.postProcessBeanDefinitionRegistry()
    │
    ▼
AutoConfigurationImportSelector.getAutoConfigurationEntry()
    │
    ▼
加载自动配置类
```

这是 BeanFactory 后置处理器阶段，在 Bean 实例化之前执行。

---

## Q2：为什么自动配置类在普通配置类之后处理？

**A**：`AutoConfigurationImportSelector` 实现了 `DeferredImportSelector` 接口：

```java
public class AutoConfigurationImportSelector
        implements DeferredImportSelector {
    // DeferredImportSelector 在所有 @Configuration 处理完成后执行
}
```

这样保证：
1. 用户代码中的 `@Configuration` 和 `@Component` 先处理
2. `@ConditionalOnMissingBean` 能正确检测到用户手动配置的 Bean
3. 用户配置优先级最高

---

## Q3：自动配置失败会报错吗？

**A**：**不会**。条件不满足时静默跳过：

```java
@ConditionalOnClass(MissingClass.class)  // MissingClass 不存在
// → 整个配置类被跳过，不报错
```

但如果配置类加载成功后内部出错：
```java
public class MyAutoConfiguration {
    @Bean
    public MyService myService() {
        throw new RuntimeException("初始化失败");
        // → 会报错，应用启动失败
    }
}
```

---

## Q4：如何追踪整个自动配置过程？

**A**：开启详细日志：

```properties
logging:
  level:
    org.springframework.boot.autoconfigure: DEBUG
    org.springframework.boot.autoconfigure.condition: TRACE
```

或在代码中：
```java
System.setProperty("spring.main.banner-mode", "console");
```

