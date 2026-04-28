# Spring Boot 配置优先级 QA

## Q1：同一个配置项，优先级高的会完全覆盖吗？

**A**：是的，完全覆盖。高优先级配置会替换低优先级的整个值，而不是合并。

---

## Q2：file:./config 和 file:. 有什么区别？

**A**：
- `file:./config/` — 和 jar 包同级目录的 config 子目录
- `file:./` — 和 jar 包同级的根目录

通常用 `config/` 目录存放公共配置。

---

## Q3：命令行参数可以传数组吗？

**A**：可以：

```bash
java -jar app.jar --server.address[0]=127.0.0.1 --server.address[1]=192.168.1.1
```

---

## Q4：Spring Boot 2.4 之后的优先级有变化吗？

**A**：2.4 之后，properties 和 YAML 同级加载，profile 文档的 `spring.config.activate.on-profile` 会影响生效时机。

---

## Q5：如何查看加载了哪些配置文件？

**A**：
```bash
java -jar app.jar --debug
```

或者查看启动日志中的 `PropertySource` 加载顺序。

---

## Q6：如何让某个配置源完全失效？

**A**：
```bash
# 排除内置配置源
java -jar app.jar --spring.config.location=none
```
