# application.properties 格式 QA

## Q1：properties 文件中文乱码怎么解决？

**A**：有三种方式：
- **方式一**：IDEA 设置 `File Encodings` 为 UTF-8
- **方式二**：手动 Unicode 转义 `\uXXXX`
- **方式三**：改用 YAML 格式，直接支持中文

---

## Q2：properties 和 YAML 同时存在，谁生效？

**A**：按 [[14_SpringBoot配置优先级]] 规则，后加载的覆盖先加载的。
- Spring Boot 2.4 之前：properties 优先级高于 YAML
- Spring Boot 2.4+：两者平级，谁后加载谁生效

---

## Q3：配置项有多个值怎么写？

**A**：使用索引 `[0]`, `[1]`...：

```properties
server.address[0]=127.0.0.1
server.address[1]=192.168.1.1
```

对应 Java List：
```java
@ConfigurationProperties
public class ServerProperties {
    private List<String> address;
}
```

---

## Q4：相同配置写多遍会怎样？

**A**：后面的会覆盖前面的（按加载顺序），但建议保持唯一性，避免维护困惑。
