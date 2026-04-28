# RandomValuePropertySource 随机值

## 先说结论

Spring Boot 提供 `${random.*}` 语法生成随机值，适合密码、密钥、测试数据等需要随机值的场景。

## 深度解析

### 基本语法

```yaml
# application.yml
app:
  # 随机整数（范围）
  token: ${random.int}
  port: ${random.int(1000,9999)}

  # 随机字符串（长度）
  secret: ${random.value}
  key: ${random.uuid}

  # 随机 long
  id: ${random.long}
  offset: ${random.long(1000000,9999999)}
```

### 随机类型

| 类型 | 语法 | 示例 |
|------|------|------|
| int | `${random.int}` | 随机整数 |
| int 范围 | `${random.int(1,100)}` | 1-100 随机整数 |
| long | `${random.long}` | 随机长整数 |
| long 范围 | `${random.long(1000,9999)}` | 1000-9999 长整数 |
| uuid | `${random.uuid}` | UUID 字符串 |
| value | `${random.value}` | 随机字符串（32字符） |

### 实际应用

```yaml
spring:
  datasource:
    # 生成随机密码
    password: ${random.value}

  # 生成随机端口（避免冲突）
  devtools:
    restart:
      port: ${random.int(10000,60000)}

app:
  # 生成随机 key
  api-key: ${random.uuid}

  # 生成随机盐值
  salt: ${random.value}
```

### 在 Java 中使用

```java
@Value("${random.int(1,100)}")
private int randomInt;

@Value("${random.uuid}")
private String uuid;
```

## 易错点/踩坑

- ❌ `${random.int}` 范围写反 — 第一个数要小于第二个数
- ❌ 随机值每次启动都变 — 测试时难以复现问题
- ❌ 生产环境使用未保存的随机值 — 密码变更后无法登录

## 关联知识点

- [[25_配置占位符]] — 占位符语法
- [[26_Relaxed属性名绑定]] — 属性绑定规则
