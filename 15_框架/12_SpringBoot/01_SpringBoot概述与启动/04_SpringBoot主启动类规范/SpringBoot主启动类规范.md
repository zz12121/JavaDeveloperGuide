# Spring Boot 主启动类规范

## 先说结论

Spring Boot 主启动类是应用的入口点，必须放在**根包**下，通常使用 `@SpringBootApplication` 注解标识，包含 `main` 方法调用 `SpringApplication.run()`。

## 深度解析

### 标准结构

```
com.example.myapp/
├── MyappApplication.java      ← 主启动类（根包下）
├── controller/
│   └── UserController.java
├── service/
│   └── UserService.java
├── mapper/
│   └── UserMapper.java
└── entity/
    └── User.java
```

### 主启动类代码

```java
package com.example.myapp;

@SpringBootApplication
public class MyappApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyappApplication.class, args);
    }
}
```

### 关键要素

| 要素 | 说明 | 注意事项 |
|------|------|----------|
| **位置** | 根包下（最外层） | 确保能扫描到所有组件 |
| **注解** | @SpringBootApplication | 核心组合注解 |
| **main 方法** | 固定格式 | 不要忘记 `static` |
| **参数** | `args` | 可传递命令行参数 |

### 为什么要在根包下

```
✅ 正确位置：
com.example.myapp/
└── MyappApplication.java
    ├── @ComponentScan 默认扫描 com.example.myapp 及其子包
    └── 所有组件都在扫描范围内

❌ 错误位置：
com.example.myapp/
├── MyappApplication.java    ← 根包
└── controller/
    └── UserController.java
    └── 如果放在子包里，该包可能扫描不到
```

### 根包扫描原理

```
@SpringBootApplication 包含 @ComponentScan

默认扫描规则：
    主启动类所在包 → com.example.myapp
    ↓
    扫描 com.example.myapp.**（所有子包）
    ↓
    注册所有 @Component/@Service/@Repository/@Controller
```

### 自定义扫描路径

如果主启动类不在根包，可以通过 `scanBasePackages` 指定：

```java
@SpringBootApplication(scanBasePackages = "com.example.myapp")
public class MyappApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyappApplication.class, args);
    }
}

// 或者指定多个包
@SpringBootApplication(scanBasePackages = {"com.example.myapp", "com.example.common"})
```

## 易错点/踩坑

- ❌ 主启动类放在子包里，导致部分组件扫描不到
- ❌ `@ComponentScan` 和 `@SpringBootApplication` 混用
- ❌ 主启动类名称随意（如 App.java、Start.java）

## 代码示例

### 标准主启动类

```java
package com.example.demo;

@SpringBootApplication
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

### 自定义配置的主启动类

```java
package com.example.demo;

@SpringBootApplication
public class DemoApplication {
    
    public static void main(String[] args) {
        // 自定义 SpringApplication 配置
        SpringApplication app = new SpringApplication(DemoApplication.class);
        
        // 关闭 Banner
        app.setBannerMode(Banner.Mode.OFF);
        
        // 设置额外的配置源
        app.setSources(Collections.singleton("classpath:extra-config.xml"));
        
        app.run(args);
    }
}
```

### 多模块项目的主启动类

```java
package com.example;

@SpringBootApplication(scanBasePackages = {
    "com.example.module1",
    "com.example.module2"
})
public class MainApplication {
    public static void main(String[] args) {
        SpringApplication.run(MainApplication.class, args);
    }
}
```

## 图解/流程

```
┌─────────────────────────────────────────────────────────┐
│                   Spring Boot 启动流程                  │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  java -jar app.jar                                      │
│       │                                                 │
│       ↓                                                 │
│  main() 方法执行                                        │
│       │                                                 │
│       ↓                                                 │
│  SpringApplication.run(Application.class, args)         │
│       │                                                 │
│       ├── 1. 创建 SpringApplication 实例               │
│       ├── 2. 加载 spring.factories                      │
│       ├── 3. 判断 Web 应用类型                          │
│       └── 4. 执行 run() 方法                           │
│               │                                         │
│               ├── 准备 Environment                      │
│               ├── 加载 ApplicationContext                │
│               ├── 执行 Runner                            │
│               └── 启动内嵌容器                           │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

## 关联知识点

- `@SpringBootApplication` 组合注解
- `@ComponentScan` 组件扫描
- `SpringApplication.run()` 启动流程
- 自动配置原理
