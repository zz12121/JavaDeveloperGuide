# 环境变量自动映射 QA

## Q1：为什么用环境变量而不是配置文件？

**A**：
- **容器化**：Docker/K8s 常用环境变量注入配置
- **安全性**：敏感信息不在代码仓库
- **灵活性**：同一镜像多环境部署

---

## Q2：如何调试环境变量映射？

**A**：
```bash
# 打印所有环境变量
printenv | grep SPRING

# 查看 Spring Boot 加载的配置源
java -jar app.jar --debug | grep "PropertySource"
```

---

## Q3：哪些字符不能用在环境变量名中？

**A**：不同系统限制不同：
- Linux：字母、数字、下划线
- Windows：字母、数字、下划线
- Docker：同 Linux

---

## Q4：环境变量中有 `-`（横线）怎么处理？

**A**：环境变量不支持 `-`，只能通过下划线映射：

```
app.config-file → app.config.file
app.config_file → app.config.file
```

---

## Q5：可以在 application.yml 中引用环境变量吗？

**A**：可以：

```yaml
spring:
  datasource:
    url: ${DB_URL}
    password: ${DB_PASSWORD:default_password}  # 带默认值
```

---

## Q6：Spring Boot 预留了哪些环境变量？

**A**：常见的有：
- `SPRING_PROFILES_ACTIVE`
- `SPRING_CONFIG_LOCATION`
- `SERVER_PORT`
- `SERVER_ERROR_PATH`
