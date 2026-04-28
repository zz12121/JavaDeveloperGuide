# RandomValuePropertySource 随机值 QA

## Q1：随机值每次启动都会变吗？

**A**：是的，`${random.*}` 在应用启动时生成新值。如果需要固定值用于测试，可以：

```yaml
# 禁用随机值（手动指定）
app:
  token: 123456  # 固定值
```

---

## Q2：可以限制随机值的范围吗？

**A**：可以：

```yaml
port: ${random.int(8000,9000)}  # 8000-9000 之间
count: ${random.int(10)}        # 0-9 之间
```

---

## Q3：${random.value} 生成的是什么？

**A**：生成一个 32 字符的随机字符串（类似 MD5），可作为密钥或盐值。

---

## Q4：生产环境适合用随机密码吗？

**A**：**不适合**。随机密码每次启动都变，会导致无法登录。正确做法：

```yaml
# 开发/测试
spring:
  datasource:
    password: ${random.value}

# 生产（外部配置）
spring:
  datasource:
    password: ${DB_PASSWORD}  # 环境变量注入
```

---

## Q5：RandomValuePropertySource 原理是什么？

**A**：Spring Boot 内置的 `PropertySource`，在配置加载时拦截 `${random.*}` 并替换为实际随机值。

---

## Q6：${random} 和 ${RANDOM} 有区别吗？

**A**：没有区别，都是 Spring Boot 的 RandomValuePropertySource。
