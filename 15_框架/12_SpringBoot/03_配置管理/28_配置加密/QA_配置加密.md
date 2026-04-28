# 配置加密 QA

## Q1：Jasypt 安全吗？

**A**：相对安全，但要注意：
- 使用强算法（如 `PBEWithHMACSHA512`）
- 密钥安全保管，不放在代码仓库
- 定期轮换密钥

---

## Q2：密钥怎么管理？

**A**：几种方式：
- 环境变量（生产环境）
- 配置中心（集中管理）
- Kubernetes Secret
- Vault

---

## Q3：Spring Boot 有内置加密吗？

**A**：Spring Boot 2.4+ 提供 `spring.cloud.bootstrap.encrypt.enabled=true`，但功能有限。推荐用专业工具。

---

## Q4：加密后配置变了，但应用读不到？

**A**：检查：
1. 密钥是否正确
2. 算法是否匹配
3. 密文格式是否正确（以 `ENC(...)` 包裹）

---

## Q5：可以只加密部分配置吗？

**A**：可以，只对需要的属性加密：

```yaml
spring:
  datasource:
    password: ENC(encrypted_value)
    username: plain_text  # 不加密
```

---

## Q6：使用加密后如何调试？

**A**：使用环境变量传入密钥，本地调试时可以临时禁用：

```java
@Bean
public static PropertySourcesPlaceholderConfigurer configurer() {
    PropertySourcesPlaceholderConfigurer configurer = new PropertySourcesPlaceholderConfigurer();
    configurer.setIgnoreUnresolvablePlaceholders(true);
    return configurer;
}
```
