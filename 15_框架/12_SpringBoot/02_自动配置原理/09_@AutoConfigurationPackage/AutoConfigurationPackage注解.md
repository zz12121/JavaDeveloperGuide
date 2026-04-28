# @AutoConfigurationPackage 注解

## 先说结论

`@AutoConfigurationPackage` 用于标记自动配置类的包路径，使得该包及其子包可以被 `@ComponentScan` 等机制扫描到。通常与 `@Import` 配合使用，将指定包注册为组件扫描的基础包。

## 深度解析

### 注解源码

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(AutoConfigurationPackages.Registrar.class)
public @interface AutoConfigurationPackage {
    String[] basePackages() default {};
    Class<?>[] basePackageClasses() default {};
}
```

### 工作原理

```java
// 自动配置包注册器
public static class Registrar implements ImportBeanDefinitionRegistrar {

    @Override
    public void registerBeanDefinitions(
            AnnotationMetadata importingClassMetadata,
            BeanDefinitionRegistry registry) {

        AnnotationAttributes attributes =
            AnnotationAttributes.fromMap(importingClassMetadata
                .getAnnotationAttributes(AutoConfigurationPackage.class.getName()));

        // 获取要注册的包名
        List<String> packages = getPackages(attributes);

        // 注册为自动配置包（后续 @ComponentScan 可以扫描这些包）
        new AutoConfigurationPackages.Packages(registry).add(packages);
    }
}
```

### 使用场景

```java
// 主启动类
@SpringBootApplication
// 等效于：
// @Configuration
// @EnableAutoConfiguration
// @ComponentScan(basePackages = "com.example.myapp")
public class MyApplication { }

// 如果有自定义自动配置包
@SpringBootApplication
@AutoConfigurationPackage(basePackages = "com.example.autoconfigure")
public class MyApplication { }
```

### 与 @ComponentScan 的关系

```java
// @AutoConfigurationPackage 会将被标记的包添加到扫描路径
@AutoConfigurationPackage
// 等效于：
// basePackages = "com.example.config"
```

## 易错点/踩坑

- ❌ **与 @SpringBootApplication 同时使用** — 可能导致重复扫描
- ❌ **basePackages 写错** — 不会报错，但扫描不到目标包

## 关联知识点

- [[SpringBootApplication组合注解]] — 主启动类注解
- [[EnableAutoConfiguration启用自动配置]] — 自动配置启用机制
