# 静态资源映射路径配置 - QA

## Q1：静态资源和视图路径可以分开吗？

**A**：可以。

```yaml
spring:
  mvc:
    static-path-pattern: /static/**
    view:
      prefix: /WEB-INF/views/
```

---

## Q2：如何映射外部文件目录？

**A**：

```java
registry.addResourceHandler("/uploads/**")
        .addResourceLocations("file:D:/uploads/");
```

---

## Q3：静态资源如何设置缓存？

**A**：

```java
registry.addResourceHandler("/static/**")
        .setCacheControl(CacheControl.maxAge(Duration.ofDays(30)));
```

---

## Q4：多个 ResourceHandler 优先级？

**A**：按添加顺序，后添加的优先匹配。
