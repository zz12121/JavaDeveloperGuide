# AutoConfiguration.imports 文件

## Q1：如何从 spring.factories 迁移到 AutoConfiguration.imports？

**A**：三步完成迁移：

**旧格式** `META-INF/spring.factories`：
```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  com.example.mystarter.MyAutoConfiguration,\
  com.example.mystarter.AnotherAutoConfiguration
```

**新格式** `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`：
```text
com.example.mystarter.MyAutoConfiguration
com.example.mystarter.AnotherAutoConfiguration
```

**步骤**：
1. 创建新路径 `META-INF/spring/`
2. 创建文件 `org.springframework.boot.autoconfigure.AutoConfiguration.imports`
3. 每行写入一个自动配置类的全限定名
4. 删除旧 `META-INF/spring.factories` 中的对应条目（或保留，Spring Boot 会优先读取 imports）

---

## Q2：spring.factories 和 imports 可以共存吗？

**A**：可以，Spring Boot 2.7 的处理顺序：

```
1. 先读取 AutoConfiguration.imports 文件
2. 再读取 spring.factories 中的 EnableAutoConfiguration
3. 合并并去重
```

**注意**：3.0+ 可能移除对 spring.factories 的支持，建议尽早迁移。

---

## Q3：AutoConfiguration.imports 支持哪些特性？

**A**：

| 特性 | 支持 | 说明 |
|------|------|------|
| `#` 注释 | ✅ | 以 `#` 开头的行视为注释 |
| 空行 | ✅ | 自动忽略 |
| 多行 | ✅ | 不需要 `\` 续行符 |
| 条件注解 | ✅ | 在配置类本身上标注，不在 imports 中 |
| 排序控制 | ✅ | 在配置类上用 `@AutoConfigureOrder` |

---

## Q4：多个 JAR 中的 imports 文件如何合并？

**A**：同 `spring.factories`，多个 JAR 中的 imports 会按 classpath 顺序追加合并：

```text
# JAR A 中的 imports
com.example.a.AutoConfigA

# JAR B 中的 imports
com.example.b.AutoConfigB

# 最终两个都加载
```

---

## Q5：AutoConfiguration.imports 中可以使用通配符吗？

**A**：不可以。必须是完整的自动配置类全限定名：

```text
# ❌ 不支持通配符
com.example.mystarter.*

# ✅ 必须逐个列出
com.example.mystarter.MyAutoConfiguration
com.example.mystarter.MySecondAutoConfiguration
```
