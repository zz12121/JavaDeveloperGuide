# AutoConfigurationImportSelector

## 先说结论

`AutoConfigurationImportSelector` 是 Spring Boot 自动配置的核心实现类，通过 `DeferredImportSelector` 机制在配置类处理阶段从配置文件加载候选自动配置类列表，并负责过滤和去重。

## 深度解析

### 类继承结构

```
AutoConfigurationImportSelector
    └── DeferredImportSelector
            └── ImportSelector
                    └── EnvironmentAware (获取环境变量)

    实现接口：
    - BeanClassLoaderAware    → 加载类
    - ResourceLoaderAware     → 加载资源
    - BeanFactoryAware        → Bean 工厂
    - EnvironmentAware        → 获取环境配置
    - Ordered                 → 排序
```

### 核心方法流程

```java
public class AutoConfigurationImportSelector
        implements DeferredImportSelector {

    // 入口方法：返回要导入的配置类
    @Override
    public String[] selectImports(AnnotationMetadata annotationMetadata) {
        AutoConfigurationEntry entry =
            getAutoConfigurationEntry(annotationMetadata);
        return toStringArray(entry.getConfigurations());
    }

    // 获取自动配置条目（核心）
    protected AutoConfigurationEntry
            getAutoConfigurationEntry(AnnotationMetadata annotationMetadata) {

        // 1. 获取启用/排除配置
        EnableAutoConfiguration ann = getAnnotationClass(annotationMetadata);
        boolean trace = logger.isTraceEnabled();
        List<String> exclusions = getExclusions(annotationMetadata, ann);

        // 2. 读取配置列表
        List<String> configurations = getCandidateConfigurations(
            annotationMetadata, ann);

        // 3. 去重
        configurations.removeAll(exclusions);

        // 4. 过滤（按条件）
        configurations = getConfigurationClassFilter()
            .filter(configurations);

        // 5. 排序
        configurations = sort(configurations);

        return new AutoConfigurationEntry(configurations, exclusions);
    }
}
```

### 配置读取来源

```java
// Spring Boot 2.7+
List<String> configurations =
    SpringFactoriesLoader.loadFactoryNames(
        getSpringFactoriesLoaderFactoryClass(),
        getBeanClassLoader()
    );
// 等效于读取：
// META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

## 易错点/踩坑

- ❌ **不要直接继承此类** — 它是 `DeferredImportSelector`，Spring 框架内部机制处理其加载
- ❌ **重写 selectImports 时忘记父类调用** — 会导致自动配置完全失效
- ❌ **getAutoConfigurationEntry 每次启动都会调用** — 不适合在此做耗时操作

## 关联知识点

- [[EnableAutoConfiguration启用自动配置]] — 此选择器的触发来源
- [[AutoConfiguration_imports文件]] — 配置文件的格式
- [[spring.factories文件]] — 旧版配置格式
