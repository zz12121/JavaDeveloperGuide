# 命令行参数配置 QA

## Q1：命令行参数和环境变量哪个优先级高？

**A**：命令行参数优先级更高（优先级 15 > 14）。

---

## Q2：参数值有特殊字符怎么办？

**A**：用引号包裹：

```bash
java -jar app.jar --app.desc="包含,特殊#字符"
```

---

## Q3：可以禁用某个配置属性吗？

**A**：某些属性支持空值禁用：

```bash
java -jar app.jar --server.port=
```

但并非所有属性都支持。

---

## Q4：Spring Boot 应用如何获取命令行参数？

**A**：两种方式：

```java
// 方式一：Spring Boot 自动绑定
public static void main(String[] args) {
    SpringApplication.run(Application.class, args);
}

// 方式二：手动获取
public static void main(String[] args) {
    System.out.println(Arrays.toString(args));
    SpringApplication.run(Application.class, args);
}
```

---

## Q5：参数中包含 `--` 怎么办？

**A**：用 `=` 而不是空格分隔：

```bash
# 错误
java -jar app.jar --name value

# 正确
java -jar app.jar --name=value
```

---

## Q6：Docker/Kubernetes 中如何传递命令行参数？

**A**：

```dockerfile
# Dockerfile
ENTRYPOINT ["java", "-jar", "app.jar"]
CMD ["--spring.profiles.active=prod"]
```

```bash
# Docker
docker run -d myapp --server.port=9090

# Kubernetes
args: ["--spring.profiles.active=prod"]
```
