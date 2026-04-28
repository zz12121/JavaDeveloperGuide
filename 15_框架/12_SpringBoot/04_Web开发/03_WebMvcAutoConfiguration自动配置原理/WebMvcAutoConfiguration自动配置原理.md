# WebMvcAutoConfiguration自动配置原理

## 先说结论

`WebMvcAutoConfiguration` 是 SpringBoot Web MVC 的自动配置核心，通过 `@ConditionalOnClass`、`@ConditionalOnMissingBean` 等条件注解实现「**用户配置优先，自动配置兜底**」的策略。

## 深度解析

### 核心概念

| 类 | 作用 |
|---|------|
| WebMvcAutoConfiguration | 主配置类 |
| WebMvcAutoConfigurationAdapter | 适配器，提供可选配置 |
| EnableWebMvcConfiguration | 继承自 `WebMvcConfigurationSupport` |
| WebMvcProperties | 绑定 `spring.mvc.*` 配置 |

### 自动配置流程

```
@EnableAutoConfiguration
        ↓
扫描 spring-boot-autoconfigure.jar
        ↓
发现 WebMvcAutoConfiguration
        ↓
检查条件注解
        ↓
所有条件满足
        ↓
执行配置
```

### 关键条件注解

| 条件 | 含义 |
|------|------|
| `@ConditionalOnClass(WebMvcConfigurer.class)` | WebMvcConfigurer 类存在时才生效 |
| `@ConditionalOnMissingBean(WebMvcConfigurer.class)` | 不存在 WebMvcConfigurer 时生效全部 |
| `@ConditionalOnMissingBean(type = "DispatcherServlet")` | DispatcherServlet 不存在时注册 |
| `@ConditionalOnProperty(prefix = "spring.mvc", name = "ignore")` | 配置关闭时才生效 |

### 配置了什么

```java
// WebMvcAutoConfiguration 主要配置
1. RequestMappingHandlerMapping      // @RequestMapping 映射
2. RequestMappingHandlerAdapter      // 请求处理
3. ExceptionHandlerExceptionResolver  // 异常处理
4. ContentNegotiationViewResolver     // 内容协商
5. HttpMessageConverterBundle         // JSON/XML 转换
6. StaticPathPatternHandlerMapping    // 静态资源
7. WelcomePageHandlerMapping         // 欢迎页
8. DispatcherServlet                  // 核心 Servlet
```

## 易错点/踩坑

- ❌ 自定义 `WebMvcConfigurer` 后静态资源失效 → 重写了 `addResourceHandlers`，检查逻辑
- ❌ 自定义 `WebMvcConfigurer` 后 JSON 返回乱码 → 重写了 `configureMessageConverters`，检查编码配置
- ❌ 404 找不到 Controller → 检查 `RequestMappingHandlerMapping` 是否正确注册

## 代码示例

### 查看自动配置报告

```bash
# 启动时添加参数，查看自动配置报告
java -jar app.jar --debug

# 或在配置文件启用
spring:
  debug: true
```

输出中搜索 `WebMvcAutoConfiguration` 查看生效情况。

## 图解/流程

```
SpringFactoriesLoader 加载
        ↓
AutoConfigurationImportSelector
        ↓
获取所有自动配置类
        ↓
WebMvcAutoConfiguration
        ↓
┌──────────────────────────────────┐
│ @ConditionalOnMissingBean        │
│   WebMvcConfigurer → 不存在      │
│   → 注册默认所有组件              │
├──────────────────────────────────┤
│ @ConditionalOnMissingBean        │
│   WebMvcConfigurer → 存在        │
│   → 只注册未自定义的组件          │
└──────────────────────────────────┘
        ↓
用户配置 + 自动配置合并
        ↓
最终生效
```

## 关联知识点

- `01_SpringBoot自动配置WebMvc`：使用层面
- `02_自动配置原理`：通用自动配置机制
