# 外部化配置 QA

## Q1：外部化配置和 Spring Boot 配置有什么关系？

**A**：外部化配置是 Spring Boot 的核心理念，application.yml/properties 都是外部化配置的具体实现。

---

## Q2：如何在不重启应用的情况下更新配置？

**A**：几种方式：
- **Spring Cloud + @RefreshScope**：手动刷新
- **Spring Boot Admin**：可视化刷新
- **配置中心**：Nacos/Apollo 等支持热更新

---

## Q3：敏感配置（如数据库密码）怎么处理？

**A**：
- 使用环境变量
- 使用配置加密（Jasypt）
- 使用配置中心（带权限控制）
- Kubernetes Secret

---

## Q4：配置中心和 application.yml 哪个优先？

**A**：配置中心通常优先级更高，应用的本地配置作为兜底。

---

## Q5：外部化配置适合哪些场景？

**A**：
- 微服务多环境部署
- Docker/Kubernetes 容器化
- 多租户系统
- 持续集成/持续部署
