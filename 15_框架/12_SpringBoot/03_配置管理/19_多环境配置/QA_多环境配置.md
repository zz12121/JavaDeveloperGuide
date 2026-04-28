# 多环境配置 QA

## Q1：profile 配置和公共配置冲突时谁生效？

**A**：**profile 配置优先**。加载顺序：
1. `application.yml`（公共）
2. `application-{profile}.yml`（profile）

后面的覆盖前面的同名属性。

---

## Q2：可以不创建公共配置吗？

**A**：可以。所有配置都可以写在 profile 文件中，但不利于维护。

---

## Q3：多个 profile 可以同时激活吗？

**A**：可以：

```bash
--spring.profiles.active=dev,common
```

会同时加载 `application-dev.yml` 和 `application-common.yml`。

---

## Q4：profile 文件名有什么要求？

**A**：
- 格式：`application-{name}.yml` 或 `application-{name}.properties`
- `{name}` 支持字母、数字、下划线
- 常用：`dev`, `test`, `prod`, `staging`

---

## Q5：如何验证当前激活的 profile？

**A**：
```java
@Autowired
private Environment environment;

System.out.println(Arrays.toString(environment.getActiveProfiles()));
```

或访问 `/actuator/env` 端点。

---

## Q6：Spring Boot 3.x 的 profile 配置有变化吗？

**A**：基本一致。3.x 支持 `spring.config.activate.on-profile` 替代传统的 profile 文件命名方式。
