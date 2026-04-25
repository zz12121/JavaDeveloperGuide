---
title: 对象序列化面试题
tags:
  - Java/IO
  - 原理型
  - 问答
module: 06_IO与NIO
created: 2026-04-18
---

# 对象序列化

## Q1：什么是 Java 序列化？使用场景有哪些？

**A：**

### 定义

Java 序列化（Serialization）是指将 **Java 对象的状态转换为字节序列**的过程，反序列化（Deserialization）则是从字节序列中**恢复 Java 对象**的过程。这使得对象可以被：

- 持久化存储到磁盘文件
- 通过网络在分布式系统间传输
- 缓存到 Redis/Memcached 等外部存储
- 在 JVM 进程间通过管道传递

### 序列化的基本用法

```java
import java.io.*;

// ===== 1. 实现 Serializable 接口（标记接口，无需实现任何方法）=====
class User implements Serializable {
    private static final long serialVersionUID = 1L;

    private String name;
    private transient String password;  // transient：不参与序列化
    private int age;

    public User(String name, String password, int age) {
        this.name = name;
        this.password = password;
        this.age = age;
    }

    @Override
    public String toString() {
        return "User{name='" + name + "', password='" + password + "', age=" + age + "}";
    }
}

public class SerializationDemo {
    public static void main(String[] args) throws Exception {
        User user = new User("张三", "secret123", 25);

        // ===== 序列化：对象 → 字节流 → 文件 =====
        try (ObjectOutputStream oos = new ObjectOutputStream(
                new FileOutputStream("user.dat"))) {
            oos.writeObject(user);   // 将整个对象写入文件
            System.out.println("序列化完成");
        }

        // ===== 反序列化：文件 → 字节流 → 对象 =====
        try (ObjectInputStream ois = new ObjectInputStream(
                new FileInputStream("user.dat"))) {
            User restored = (User) ois.readObject();  // 从字节恢复对象
            System.out.println("反序列化：" + restored);
            // 输出: User{name='张三', password='null', age=25}
            // 注意：password 是 null！因为标记了 transient
        }
    }
}
```

### 核心流程图解

```
序列化过程：
┌──────────┐    ObjectOutputStream     ┌─────────┐    FileOutputStream    ┌───────┐
│ Java对象  │ ──→ (反射获取字段值) ──→ │字节序列  │ ──→ (写出到OS) ──→      │ 磁盘  │
│(User实例) │                           │         │                        │文件   │
└──────────┘                           └─────────┘                        └───────┘

反序列化过程：
┌───────┐    FileInputStream     ┌─────────┐    ObjectInputStream    ┌──────────┐
│ 磁盘   │ ──→ (读取字节) ──→    │字节序列  │ ──→ (反射创建对象+赋值) ──→│Java对象  │
│文件   │                         │         │                          │(User实例)│
└───────┘                         └─────────┘                          └──────────┘
```

### 常见使用场景

| **场景** | **说明** | **示例** |
|----------|---------|---------|
| **持久化存储** | 将对象保存到磁盘 | 游戏存档、配置对象缓存 |
| **网络传输** | RPC 远程调用传参/返回值 | Dubbo、gRPC（部分协议）、RMI |
| **分布式缓存** | 将对象存入 Redis | Spring Cache + Redis |
| **深拷贝** | 利用序列化/反序列化复制对象 | 需要完全独立副本时 |
| **Session 复制** | Web集群间同步会话 | Tomcat Cluster Session Replication |
| **消息队列** | 对象消息体传输 | ActiveMQ ObjectMessage |

---

## Q2：Serializable 和 Externalizable 的区别？

**A：**

### 详细对比

| **维度** | **Serializable** | **Externalizable** |
|----------|-----------------|-------------------|
| **接口类型** | **标记接口**（无方法） | **普通接口**（有2个方法） |
| **控制粒度** | JVM 全权控制 | 开发者**完全控制** |
| **序列化方式** | 反射自动序列化所有 `non-static`、`non-transient` 字段 | 手动实现 `writeExternal/readExternal` |
| **性能** | 较低（反射开销 + 序列化所有字段） | 更高（只序列化需要的字段，无反射） |
| **构造器要求** | 不需要特殊构造器 | **必须有无参构造器**（反序列化时调用） |
| **灵活性** | 低（要么全序列化，要么用 transient 排除） | 高（可自定义任意逻辑） |
| **serialVersionUID** | 可选（不写JVM自动生成，但风险大） | 不需要（版本控制自己管理） |
| **适用场景** | 大多数常规场景、快速开发 | 高性能需求、敏感数据过滤 |

### Externalizable 完整示例

```java
import java.io.*;
import java.util.Base64;

public class ExternalizableDemo {

    /**
     * 使用 Externalizable 自定义序列化逻辑
     * 典型应用：密码加密后再序列化
     */
    static class SecureUser implements Externalizable {
        private String username;       // 明文序列化
        private transient String password; // 不参与默认序列化
        private int loginCount;        // 明文序列化
        private Date lastLogin;        // 明文序列化

        // ⚠️ 必须提供无参构造器！否则抛 InvalidClassException
        public SecureUser() {  // 反序列化时JVM通过反射调用此构造器
        }

        public SecureUser(String username, String password) {
            this.username = username;
            this.password = password;
            this.loginCount = 0;
            this.lastLogin = new Date();
        }

        @Override
        public void writeExternal(ObjectOutput out) throws IOException {
            // ⭐ 完全由开发者控制写什么、怎么写
            out.writeUTF(username);
            out.writeInt(loginCount);
            out.writeLong(lastLogin.getTime());
            // 密码：Base64编码后写出（不是明文！）
            String encodedPwd = Base64.getEncoder().encodeToString(password.getBytes());
            out.writeUTF(encodedPwd);
            // 可以添加版本号用于后续兼容性处理
            out.writeInt(1); // protocol version
        }

        @Override
        public void readExternal(ObjectInput in) throws IOException, ClassNotFoundException {
            // ⭐ 读取顺序必须与 writeExternal 完全一致！
            username = in.readUTF();
            loginCount = in.readInt();
            lastLogin = new Date(in.readLong());
            String encodedPwd = in.readUTF();
            password = new String(Base64.getDecoder().decode(encodedPwd));
            int version = in.readInt();
            if (version > 1) {
                // 未来可以在这里做兼容性处理
            }
        }

        @Override
        public String toString() {
            return String.format("SecureUser{user=%s, pwd=%s, count=%d}",
                    username, "***", loginCount);
        }
    }

    public static void main(String[] args) throws Exception {
        SecureUser user = new SecureUser("admin", "P@ssw0rd");
        user.loginCount = 42;

        // 序列化
        try (ObjectOutputStream oos = new ObjectOutputStream(
                new FileOutputStream("secure_user.dat"))) {
            oos.writeObject(user);
        }
        System.out.println("序列化大小: " + Files.size(Paths.get("secure_user.dat")) + " bytes");

        // 反序列化
        try (ObjectInputStream ois = new ObjectInputStream(
                new FileInputStream("secure_user.dat"))) {
            SecureUser restored = (SecureUser) ois.readObject();
            System.out.println("反序列化: " + restored);
        }
    }
}
```

### 性能对比

```java
// 同样的数据量，Externalizable 通常比 Serializable 快 20%~50%
// 因为：
// 1. Serializable 使用反射获取所有字段（getField/get）
// 2. Serializable 包含类描述信息（类名、字段签名等元数据）
// 3. Externalizable 直接调用 writeUTF/writeInt 等基本方法

// Serializable 输出的额外元数据：
// - 类描述符（类名 + serialVersionUID）
// - 每个字段的类型和名称
// - 对象引用关系图（handle机制避免循环引用）

// Externalizable 输出的是纯数据（你写了什么就输出什么）
```

### 如何选择？

> **优先用 Serializable**（简单可靠），只在以下情况考虑 Externalizable：
> - 对性能有极高要求
> - 部分字段包含敏感信息需要加密/脱敏
> - 需要跨版本兼容（不同版本的类有字段差异）
> - 部分字段计算成本高，不需要持久化

---

## Q3：serialVersionUID 有什么作用？为什么建议显式定义？

**A：**

### 作用

`serialVersionUID`（简称 SUID）是序列化时的**版本号**，用于在反序列化时验证发送方和接收方的类定义是否一致。

### 工作原理

```
序列化时：
  ObjectOutputStream 将类的 serialVersionUID 写入字节流的头部
  （如果未显式定义，JVM根据类结构自动生成一个hash值）

反序列化时：
  ObjectInputStream 从字节流读取 SUID
  与当前 JVM 中该类的 SUID 进行比对
  ├── 一致 → 正常反序列化
  └── 不一致 → 抛出 InvalidClassException！！！
```

### 为什么必须显式定义？

```java
// ❌ 危险：不定义 serialVersionUID
class User implements Serializable {
    // 没有 static final long serialVersionUID 字段
    private String name;
}

// 问题：JVM 会根据类的结构（字段名、字段类型、修饰符等）
// 自动生成一个 hash 值作为 SUID
// 一旦你修改了类的任何结构（新增字段、改变类型等），
// 生成的 SUID 就变了！导致之前序列化的文件全部无法反序列化！

// ✅ 安全：始终显式定义
class User implements Serializable {
    private static final long serialVersionUID = 1L;  // 显式定义
    private String name;

    // 后来新增了字段...
    // private String email;  // 新增字段
    // serialVersionUID 保持为 1L 不变
    // 反序列化旧数据时：email 为 null（新字段容忍缺失）
}
```

### SUID 不一致的典型场景

```java
// ===== 版本1 =====
class Config implements Serializable {
    private static final long serialVersionUID = 1L;
    private String serverHost;
    private int serverPort;
}

// 序列化了 v1 的 Config 对象到 config_v1.dat

// ===== 版本2：修改了类结构 =====
class Config implements Serializable {
    private static final long serialVersionUID = 2L;  // 改成了2L！

    private String host;          // 重命名字段
    private int port;             // 重命名字段
    private boolean sslEnabled;   // 新增字段

    // 如果此时尝试反序列化 config_v1.dat：
    // java.io.InvalidClassException:
    //   Config; local class incompatible:
    //   stream classdesc serialVersionUID = 1,
    //   local class serialVersionUID = 2
}
```

### 最佳实践

```java
/**
 * 1. 所有实现了 Serializable 的类都应该声明 serialVersionUID
 * 2. 初始值为 1L
 * 3. 当发生"破坏性"的类变更时才递增（删除字段、改变字段类型）
 * 4. 新增字段通常不需要改 SUID（旧数据中该字段为null/0）
 * 5. IDE（IDEA/Eclipse）可以一键生成：Alt+Insert → serialVersionUID
 */
public class BestPractice implements Serializable {
    // 固定格式：private static final long serialVersionUID = <数字>L;
    private static final long serialVersionUID = 8555722949983909686L;

    private String data;
    // ...
}
```

### IDEA 自动生成设置

`Settings → Editor → Inspections → Java → Serialization issues → 勾选 'Serializable class without serialVersionUID'`

这样每次创建 Serializable 类但忘记加 SUID 时，IDEA 会给出警告。

---

## Q4：哪些字段不参与序列化？transient 关键字的详细用法？

**A：**

### 不参与序列化的字段

| **类型** | **关键字** | **原因** | **反序列化后的值** |
|----------|-----------|---------|------------------|
| `static` 字段 | —（静态修饰符） | 属于类级别，不属于对象 | 当前 JVM 中的静态变量值 |
| `transient` 字段 | `transient` | 显式标记跳过 | **默认值**（null / 0 / false） |

### transient 详细示例

```java
import java.io.*;
import java.time.LocalDate;

class Employee implements Serializable {
    private static final long serialVersionUID = 1L;

    private String name;              // ✅ 参与序列化
    private double salary;            // ✅ 参与序列化
    private transient String password; // ❌ 跳过（敏感信息不应落盘）
    private transient Thread thread;  // ❌ 跳过（Thread不可序列化！）
    private transient Connection dbConn; // ❌ 跳过（连接不可序列化）
    private static int totalEmployees; // ❌ 跳过（static属于类）

    public Employee(String name, double salary, String pwd) {
        this.name = name;
        this.salary = salary;
        this.password = pwd;
        Employee.totalEmployees++;
    }
}

public class TransientDemo {
    public static void main(String[] args) throws Exception {
        Employee emp = new Employee("Alice", 50000.0, "abc123");

        System.out.println("序列化前: name=" + emp.getName()
                + ", pwd=" + emp.getField("password")
                + ", static=" + Employee.totalEmployees);

        // 序列化
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(baos);
        oos.writeObject(emp);

        // 修改 static 值（模拟不同的JVM环境）
        Employee.totalEmployees = 999;

        // 反序列化
        ObjectInputStream ois = new ObjectInputStream(
            new ByteArrayInputStream(baos.toByteArray()));
        Employee restored = (Employee) ois.readObject();

        System.out.println("反序列化后: name=" + restored.getName()
                + ", pwd=" + restored.getField("password")  // null!
                + ", static=" + Employee.totalEmployees);    // 999! 不是原来的值
    }
}
```

### transient 的常见使用场景

#### 场景1：敏感信息安全

```java
public class CreditCard implements Serializable {
    private static final long serialVersionUID = 1L;
    private String cardNumber;           // 已脱敏的卡号
    private transient String cvv;       // CVV安全码绝不序列化
    private transient String rawPan;    // 完整卡号绝不序列化
}
```

#### 场景2：不可序列化的字段

```java
import java.sql.Connection;
import java.util.concurrent.ExecutorService;

public class ServiceContext implements Serializable {
    private static final long serialVersionUID = 1L;
    private String configId;

    // 以下都是无法序列化的类型！必须标记 transient
    private transient Connection dbConnection;      // JDBC连接
    private transient ExecutorService threadPool;   // 线程池
    private transient Logger logger;                // 日志框架的Logger
}
```

#### 场景3：可计算的派生字段

```java
public class Triangle implements Serializable {
    private static final long serialVersionUID = 1L;
    private double a, b, c;  // 三条边长

    // 面积可以根据三边计算出来，不需要序列化
    private transient Double area;  // 懒计算

    public double getArea() {
        if (area == null) {
            // 海伦公式
            double s = (a + b + c) / 2;
            area = Math.sqrt(s * (s-a) * (s-b) * (s-c));
        }
        return area;
    }

    // 反序列化后 area 为 null，首次调用 getArea() 时重新计算即可
}
```

#### 场景4：缓存字段

```java
public class UserProfile implements Serializable {
    private static final long serialVersionUID = 1L;
    private long userId;
    private String nickname;

    // 缓存的计算结果，反序列化后可重建
    private transient String avatarUrl;      // 头像URL（可重新查询）
    private transient List<String> tags;     // 标签列表（可重新查询）
    private transient LocalDateTime lastRefreshTime; // 最后刷新时间
}
```

### ⚠️ 注意事项

1. **`static` 字段虽然不序列化，但不要依赖它传递状态**
2. **`transient` 字段反序列化后为默认值**，需要在代码中妥善处理（如懒加载重建）
3. **如果父类没有实现 Serializable 但子类实现了，父类的字段不会序列化**（除非父类也有无参构造器）
4. **`final + transient` 组合是合法的**：final 字段在构造器中初始化后不可变，transient 表示不序列化——但反序列化后该字段仍保持构造器中的初始值（因为final不会被重新赋值）

---

## Q5：Java 序列化 vs JSON 序列化，如何选择？

**A：**

### 全面对比

| **对比维度** | **Java原生序列化** | **JSON序列化（Jackson/Fastjson/Gson）** |
|-------------|-------------------|--------------------------------------|
| **输出格式** | 二进制字节流 | 文本（JSON字符串） |
| **可读性** | ❌ 不可读 | ✅ 人眼可读，方便调试 |
| **体积** | 较小（二进制压缩） | 较大（文本+字段名重复） |
| **速度** | 中等（反射+元数据开销） | 取决于库（Jackson ≈ Java序列化，Fastjson2更快） |
| **语言无关性** | ❌ 仅Java生态 | ✅ 跨语言通用 |
| **安全性** | ⚠️ 存在已知漏洞（CVE） | 相对更安全（但仍需注意） |
| **灵活性** | 低（类变更易不兼容） | 高（忽略未知字段、支持缺省值） |
| **依赖** | JDK自带（零依赖） | 需要第三方库 |
| **Spring支持** | 支持（HttpMessageConverter） | 默认首选（MappingJackson2HttpMessageConverter） |

### 代码对比

```java
// ========== Java 原生序列化 ==========
class Data implements Serializable {
    private static final long serialVersionUID = 1L;
    private String name;
    private int value;
    // getter/setter...
}

// 序列化
Data data = new Data("test", 100);
ByteArrayOutputStream baos = new ByteArrayOutputStream();
new ObjectOutputStream(baos).writeObject(data);
byte[] javaBytes = baos.toByteArray();

// 反序列化
Data restored = (Data) new ObjectInputStream(
    new ByteArrayInputStream(javaBytes)).readObject();


// ========== JSON 序列化（Jackson）==========
// 无需实现任何接口！
class JsonData {
    private String name;
    private int value;
    // getter/setter（或使用注解 @JsonProperty）
}

ObjectMapper mapper = new ObjectMapper();

// 序列化
JsonData jdata = new JsonData("test", 100);
String jsonStr = mapper.writeValueAsString(jdata);
// {"name":"test","value":100}

// 反序列化
JsonData jrestored = mapper.readValue(jsonStr, JsonData.class);


// ========== JSON 序列化（Fastjson2）==========
String fastjson = com.alibaba.fastjson2.JSON.toJSONString(jdata);
JsonData fjrestored = com.alibaba.fastjson2.parseObject(fastjson, JsonData.class);
```

### 为什么现代项目大多选择 JSON 而非 Java 序列化？

#### 原因1：跨语言互操作

```java
// 微服务架构下，服务可能用不同语言编写：
// Java服务 ←JSON→ Go服务 ←JSON→ Python服务 ←JSON→ Node.js服务
// Java序列化只能在Java之间通信，限制了技术选型自由度
```

#### 原因2：安全性问题

```java
// Java 序列化存在严重的安全漏洞：
// 攻击者可以通过精心构造的字节流，在反序列化时执行任意代码
// 著名的 CVE-2015-6420、Apache Commons Collections 反序列化漏洞等

// 防御措施：
// 1. 尽量不使用 Java 序列化做外部接口
// 2. 如果必须使用，设置白名单：
ObjectInputFilter filter = ObjectInputFilter.allowFilter(
    filterInfo -> {
        Class<?> clazz = filterInfo.serialClass();
        if (clazz == null) return Status.UNDECIDED;
        return (clazz == Data.class || clazz == String.class)
               ? Status.ALLOWED : Status.REJECTED;
    }, ois.setObjectInputFilter(filter);
```

#### 原因3：版本兼容性

```java
// JSON天然支持字段增减：
// {"name":"test","value":100,"newField":"extra"}
// 旧版类只有name/value → newField被自动忽略（不会报错！）

// Java 序列化则严格得多：
// 新增字段 → 需要管理 serialVersionUID
// 删除字段/改类型 → 直接报 InvalidClassException
```

### 何时仍然应该使用 Java 序列化？

1. **纯 Java 内部通信**（如 RMI、同一 JVM 内的深拷贝）
2. **对体积敏感的场景**（二进制比文本小 30%~60%）
3. **遗留系统对接**（已有大量 Java 序列化数据文件）
4. **需要完整保留对象类型信息**（包括泛型参数等运行时类型信息）

### 性能基准参考（大致数量级）

```
操作                  Java序列化     Jackson       Fastjson2     Protobuf
─────────           ──────────     ───────       ─────────     ───────
序列化速度(op/s)      ~80K          ~120K         ~300K         ~800K
反序列化速度(op/s)    ~40K          ~100K         ~250K         ~600K
序列化后体积(相对)      1.0x          1.8x          1.5x         0.3x
```

> **结论**：新项目推荐 **JSON（Jackson/Fastjson2）** 或 **Protobuf**（极致性能）。Java 原生序列化仅在特定场景下使用。

---

## Q6：序列化过程中的单例破坏问题如何解决？

**A：**

### 问题背景

Java 序列化机制提供了一个特殊的"后门"——反序列化时会**绕过构造方法直接创建对象**。这意味着：

1. 单例模式的私有构造器被绕过
2. 枚举单例也可能受影响（虽然枚举相对安全）
3. 可能创建出多个"单例"实例

### 演示问题

```java
import java.io.*;

// 经典的双重检查锁单例
class UnsafeSingleton implements Serializable {
    private static final long serialVersionUID = 1L;

    private static volatile UnsafeSingleton instance;

    private UnsafeSingleton() {
        System.out.println("UnsafeSingleton 构造器被调用！");
    }

    public static UnsafeSingleton getInstance() {
        if (instance == null) {
            synchronized (UnsafeSingleton.class) {
                if (instance == null) {
                    instance = new UnsafeSingleton();
                }
            }
        }
        return instance;
    }
}

public class SingletonBreakDemo {
    public static void main(String[] args) throws Exception {
        // 正常获取单例
        UnsafeSingleton s1 = UnsafeSingleton.getInstance();
        UnsafeSingleton s2 = UnsafeSingleton.getInstance();
        System.out.println(s1 == s2);  // true ✓

        // 序列化再反序列化
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        new ObjectOutputStream(baos).writeObject(s1);

        UnsafeSingleton s3 = (UnsafeSingleton) new ObjectInputStream(
                new ByteArrayInputStream(baos.toByteArray())).readObject();

        System.out.println(s1 == s3);  // false ✗！产生了新的实例！
        // 单例被破坏了！
    }
}
```

### 解决方案1：枚举单例（最推荐 ★★★）

```java
/**
 * Effective Java 推荐的方式：枚举单例天然防序列化攻击
 * JVM 保证枚举实例的唯一性，反序列化时返回同一个枚举常量
 */
enum SafeSingleton {
    INSTANCE;

    private String config;

    public void setConfig(String config) { this.config = config; }
    public String getConfig() { return config; }

    // 可以像普通类一样添加方法
    public void doSomething() { /* ... */ }
}

// 测试：枚举单例不会被序列化破坏
SafeSingleton e1 = SafeSingleton.INSTANCE;
e1.setConfig("test-config");

ByteArrayOutputStream baos = new ByteArrayOutputStream();
new ObjectOutputStream(baos).writeObject(e1);

SafeSingleton e2 = (SafeSingleton) new ObjectInputStream(
    new ByteArrayInputStream(baos.toByteArray())).readObject();

System.out.println(e1 == e2);  // true ✓ 永远是同一个实例
System.out.println(e2.getConfig()); // "test-config" 状态也保留了
```

### 解决方案2：readResolve() 方法

```java
class FixedSingleton implements Serializable {
    private static final long serialVersionUID = 1L;

    private static volatile FixedSingleton INSTANCE = new FixedSingleton();

    private FixedSingleton() {}

    public static FixedSingleton getInstance() {
        return INSTANCE;
    }

    /**
     * ⭐ 反序列化时JVM会调用此方法，用其返回值替代反序列化创建的对象
     * 这就是"拦截"反序列化的关键钩子
     */
    private Object readResolve() {
        System.out.println("readResolve 被调用，返回唯一实例");
        return getInstance();  // 始终返回同一个单例实例
    }
}

// 测试
FixedSingleton f1 = FixedSingleton.getInstance();
// ...序列化和反序列化过程同上...
FixedSingleton f3 = (FixedSingleton) new ObjectInputStream(...).readObject();
System.out.println(f1 == f3);  // true ✓ readResolve保住了单例
```

### `readResolve()` 工作原理

```
正常反序列化流程：
ObjectInputStream.readObject()
  → 通过反射调用无参构造器创建新对象
  → 读取字段并赋值
  → 返回新对象

有 readResolve 的流程：
ObjectInputStream.readObject()
  → 通过反射调用无参构造器创建新对象
  → 读取字段并赋值
  → 检查是否有 readResolve() 方法？
    → 有！调用 readResolve()
    → 用 readResolve() 的返回值作为最终结果（丢弃刚才创建的对象）
    → 返回 readResolve() 的返回值
```

### 解决方案3：不实现 Serializable

最根本的解决方案——如果你的单例根本不需要序列化，那就**不要实现 Serializable**。很多情况下单例确实不应该被序列化（比如线程池、数据库连接池管理器等持有资源引用的单例）。

---

## Q7：父子类序列化有什么规则？父类没实现 Serializable 怎么办？

**A：**

### 规则总结

```
情况1：父类实现了 Serializable
  → 子类自动具备序列化能力（无需额外声明）
  → 子类新增字段也会被序列化

情况2：父类没有实现 Serializable，子类实现了
  → 子类自身的字段会被序列化
  → 父类的字段不会被序列化！
  → 要求：父类必须有**无参构造器**（反序列化时调用）

情况3：父类和子类都没实现 Serializable
  → 无法序列化
```

### 示例代码

```java
import java.io.*;

// ===== 父类：没有实现 Serializable =====
class Animal {
    protected String species;  // 此字段不会被序列化！

    // ⚠️ 必须有无参构造器！否则子类反序列化时报 InvalidClassException
    public Animal() {
        this.species = "unknown";
    }

    public Animal(String species) {
        this.species = species;
    }
}

// ===== 子类：实现了 Serializable =====
class Dog extends Animal implements Serializable {
    private static final long serialVersionUID = 1L;
    private String name;      // 此字段会被序列化
    private int age;          // 此字段会被序列化

    public Dog() {  // 同样需要无参构造器（隐式调用super()）
        super();    // 调用Animal的无参构造器
    }

    public Dog(String species, String name, int age) {
        super(species);
        this.name = name;
        this.age = age;
    }

    @Override
    public String toString() {
        return String.format("Dog{species='%s', name='%s', age=%d}",
                species, name, age);
    }
}

public class InheritanceSerializationDemo {
    public static void main(String[] args) throws Exception {
        Dog dog = new Dog("Canis lupus", "旺财", 3);
        System.out.println("原始: " + dog);
        // 原始: Dog{species='Canis lupus', name='旺财', age=3}

        // 序列化
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        new ObjectOutputStream(baos).writeObject(dog);

        // 反序列化
        Dog restored = (Dog) new ObjectInputStream(
                new ByteArrayInputStream(baos.toByteArray())).readObject();
        System.out.println("还原: " + restored);
        // 还原: Dog{species='unknown', name='旺财', age=3}
        // ↑ species 变回了 "unknown"！因为它是父类字段，未被序列化
    }
}
```

### 最佳实践

```java
// 推荐：让父类也实现 Serializable
abstract class BaseEntity implements Serializable {
    private static final long serialVersionUID = 1L;

    protected Long id;          // 公共ID字段
    protected LocalDateTime createTime;  // 公共时间字段
    // 这些字段都会被子类正确序列化
}

class User extends BaseEntity {
    private static final long serialVersionUID = 2L;
    private String username;
    // id 和 createTime 继承自BaseEntity，也能正确序列化
}
```

> 如果父类确实不能或不应实现 Serializable（如继承自第三方库的类），确保父类有无参构造器，并在子类中自行处理父类字段的恢复逻辑（可通过自定义 `writeObject/readObject` 方法手动保存和恢复父类字段值）。
