# 文件上传配置与原理 - QA

## Q1：文件大小限制在哪里配置？

**A**：

```yaml
spring:
  servlet:
    multipart:
      max-file-size: 10MB
      max-request-size: 100MB
```

**注意**：Spring Boot 2.x 使用新格式。

---

## Q2：上传失败返回什么错误？

**A**：

| 错误 | 原因 |
|------|------|
| 400 Bad Request | 未设置 `multipart/form-data` |
| 413 Payload Too Large | 文件超过限制 |
| 500 Internal Error | 临时目录不存在 |

---

## Q3：如何自定义 MultipartResolver？

**A**：

```java
@Bean
public MultipartResolver multipartResolver() {
    StandardServletMultipartResolver resolver = new StandardServletMultipartResolver();
    resolver.setMaxInMemorySize(1024);
    resolver.setMaxUploadSize(1024 * 1024 * 10);
    return resolver;
}
```

---

## Q4：Spring Boot 2.x 使用哪个 MultipartResolver？

**A**：`StandardServletMultipartResolver`（基于 Servlet 3.0+）。

**注意**：不能使用 `CommonsMultipartResolver`（需要 Apache Commons FileUpload）。
