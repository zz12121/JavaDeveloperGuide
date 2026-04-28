# Spring Boot 主启动类规范

## Q1：Spring Boot 主启动类必须放在根包下吗？为什么？

**A**：**是的，推荐放在根包下**。

**原因**：`@SpringBootApplication` 包含 `@ComponentScan`，默认扫描主启动类所在包及其子包。

```java
@SpringBootApplication
// 等价于
@Configuration
@EnableAutoConfiguration
@ComponentScan  // 默认扫描主启动类所在包
```

**示例**：
```
项目结构：
com.example.myapp/
├── MyappApplication.java    ← 主启动类在根包
├── controller/
│   └── UserController.java  ← 被扫描到 ✓
├── service/
│   └── UserService.java     ← 被扫描到 ✓
└── entity/
    └── User.java            ← 被扫描到 ✓
```

**如果主启动类在子包**：
```
com.example.myapp/
├── MyappApplication.java      ← 主启动类在根包
└── controller/
    └── UserController.java  ← 被扫描到 ✓
```

**如果主启动类放在 controller 包**：
```
com.example.myapp/
├── MainApplication.java      ← 在 controller 包 ❌
└── service/                 ← service 包不被扫描 ❌
    └── UserService.java
```

---

## Q2：主启动类的命名规范？

**A**：Spring Boot 对命名没有强制要求，但有最佳实践：

| 命名方式 | 说明 | 推荐度 |
|----------|------|--------|
| `Application` | 最简洁 | ⭐⭐⭐⭐⭐ |
| `{项目名}Application` | 明确所属 | ⭐⭐⭐⭐ |
| `App` / `Start` | 过于随意 | ⭐ |

**示例**：
```java
// ✅ 推荐
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}

// ✅ 明确项目名称
@SpringBootApplication
public class UserManagementApplication {
    public static void main(String[] args) {
        SpringApplication.run(UserManagementApplication.class, args);
    }
}
```

---

## Q3：主启动类可以有多余的配置吗？

**A**：可以，但 `@SpringBootApplication` 注解本身已经包含了大部分配置，通常不需要额外添加。

**常见需求及解决方案**：

| 需求 | 方案 | 示例 |
|------|------|------|
| 关闭 Banner | SpringApplication API | `app.setBannerMode(Banner.Mode.OFF)` |
| 关闭自动配置 | @SpringBootApplication 参数 | `exclude = {...}` |
| 指定扫描包 | @SpringBootApplication 参数 | `scanBasePackages = {...}` |
| 禁用某特性 | application.yml | `spring.main.xxx=false` |

**示例 - 自定义 Banner**：
```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication app = new SpringApplication(Application.class);
        app.setBannerMode(Banner.Mode.CONSOLE);  // 或 OFF
        app.run(args);
    }
}
```

**示例 - 排除自动配置**：
```java
@SpringBootApplication(exclude = {
    DataSourceAutoConfiguration.class,
    HibernateJpaAutoConfiguration.class
})
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

---

## Q4：主启动类的 main 方法为什么要传递 args？

**A**：`args` 参数用于接收**命令行参数**，这些参数会被 Spring Boot 使用。

**命令行参数示例**：
```bash
java -jar app.jar --server.port=9090 --spring.profiles.active=dev
```

**args 的用途**：
1. **配置属性**：如 `--server.port`
2. **激活 Profile**：`--spring.profiles.active=dev`
3. **自定义参数**：可通过 `@Value` 或 `Environment` 获取

**代码中获取命令行参数**：
```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        // 方式一：通过 Environment 获取
        ConfigurableApplicationContext ctx = SpringApplication.run(Application.class, args);
        String port = ctx.getEnvironment().getProperty("server.port");
        System.out.println("启动端口：" + port);
        
        // 方式二：通过 ApplicationArguments 获取
        SpringApplication app = new SpringApplication(Application.class);
        app.setAddArguments(ac -> {
            System.out.println("命令行参数：" + Arrays.toString(ac.getSourceArgs()));
        });
    }
}
```

---

## Q5：为什么主启动类不能直接 @Autowired？

**A**：主启动类的 `main` 方法是静态的，不属于 Spring 管理的 Bean，无法使用依赖注入。

**错误示例**：
```java
@SpringBootApplication
public class Application {
    
    @Autowired  // ❌ 错误，静态方法不能注入
    private static UserService userService;
    
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

**正确做法**：
```java
@SpringBootApplication
public class Application {
    
    public static void main(String[] args) {
        // Spring 容器在此创建
        ConfigurableApplicationContext ctx = SpringApplication.run(Application.class, args);
        
        // 从容器中获取 Bean
        UserService userService = ctx.getBean(UserService.class);
        userService.doSomething();
    }
}
```

---

## Q6：多模块项目如何设置主启动类？

**A**：每个可运行的模块都需要一个主启动类。

**项目结构示例**：
```
parent-project/
├── pom.xml
├── module-user/              ← 用户模块
│   └── src/main/java/
│       └── com.example.user/
│           └── UserApplication.java    ← 主启动类
├── module-order/             ← 订单模块
│   └── src/main/java/
│       └── com.example.order/
│           └── OrderApplication.java   ← 主启动类
└── module-common/            ← 公共模块（无启动类）
```

**module-user 的主启动类**：
```java
package com.example.user;

@SpringBootApplication(scanBasePackages = {
    "com.example.user",
    "com.example.common"  // 扫描公共模块
})
public class UserApplication {
    public static void main(String[] args) {
        SpringApplication.run(UserApplication.class, args);
    }
}
```
