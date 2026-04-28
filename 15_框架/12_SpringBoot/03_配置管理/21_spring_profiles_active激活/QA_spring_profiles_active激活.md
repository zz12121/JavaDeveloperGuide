# spring.profiles.active 激活 QA

## Q1：spring.profiles.active 和 spring.profiles.include 有什么区别？

**A**：

| 属性 | 说明 |
|------|------|
| `spring.profiles.active` | 激活指定 profile |
| `spring.profiles.include` | 额外包含 profile |

```yaml
# 激活 dev，同时包含 common
spring:
  profiles:
    active: dev
    include: common
```

2.4+ 推荐用 `add` 替代 `include`。

---

## Q2：命令行参数和环境变量哪个优先级高？

**A**：命令行参数优先级更高。

优先级：命令行 > 环境变量 > 配置文件

---

## Q3：可以禁用某个 profile 吗？

**A**：可以：

```bash
java -jar app.jar --spring.profiles.active=dev --spring.profiles.active-override=true
```

---

## Q4：profile 名称支持中文吗？

**A**：支持，但不推荐。profile 名称建议只用字母、数字、下划线。

---

## Q5：如何查看当前激活的 profile？

**A**：启动日志或代码：

```java
@Autowired
private Environment env;
System.out.println(Arrays.toString(env.getActiveProfiles()));
```

---

## Q6：可以在 application-{profile}.yml 中再设置 active 吗？

**A**：可以，但不推荐，可能导致循环引用。
