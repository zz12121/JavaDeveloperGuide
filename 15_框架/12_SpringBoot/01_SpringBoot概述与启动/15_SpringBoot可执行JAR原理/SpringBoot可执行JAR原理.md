# Spring Boot 可执行 JAR 原理

## 先说结论

Spring Boot 通过自定义 `SpringBootLoader` 类加载器，将应用和依赖打包成可执行 JAR，支持 `java -jar` 直接运行，核心是利用 JAR 的嵌套机制。

## 深度解析

### 可执行 JAR 结构

```
myapp.jar
│
├── META-INF/
│   └── MANIFEST.MF        ← 包含 Main-Class 和 Start-Class
│
├── org/springframework/boot/loader/  ← Spring Boot Loader
│   ├── JarLauncher.class
│   ├── MainLauncher.class
│   └── ...
│
└── BOOT-INF/
    ├── classes/           ← 应用的字节码
    │   └── com/example/
    │       └── Application.class
    │
    └── lib/               ← 依赖的 JAR
        ├── spring-boot-3.2.0.jar
        ├── spring-core-6.1.0.jar
        └── ...
```

### MANIFEST.MF 内容

```properties
Manifest-Version: 1.0
Spring-Boot-Classpath-Index: BOOT-INF/classpath.idx
Spring-Boot-Classes: BOOT-INF/classes/
Spring-Boot-Lib: BOOT-INF/lib/
Start-Class: com.example.Application        ← 实际 main 方法
Spring-Boot-Loader-Version: 3.2.0
Spring-Boot-Version: 3.2.0
Main-Class: org.springframework.boot.loader.JarLauncher  ← 启动器
```

### 启动流程

```
┌─────────────────────────────────────────────────────────────────┐
│                    可执行 JAR 启动流程                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   java -jar myapp.jar                                           │
│         │                                                       │
│         ↓                                                       │
│   JVM 加载 org.springframework.boot.loader.JarLauncher          │
│         │                                                       │
│         ↓                                                       │
│   JarLauncher.main()                                            │
│         │                                                       │
│         ├── 创建自定义 ClassLoader（LaunchedURLClassLoader）   │
│         │                                                       │
│         ├── 加载 BOOT-INF/classes 和 BOOT-INF/lib 中的资源    │
│         │                                                       │
│         ├── 加载 Start-Class（Application）                    │
│         │                                                       │
│         └── 调用 Application.main()                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 嵌套 JAR 机制

**问题**：标准 JAR 规范要求 JAR 中的 JAR 无法被直接加载。

**解决方案**：Spring Boot 使用自定义文件系统（LayeredFileSystemJarEntry）

```
传统 JAR（不可用）：
myapp.jar
└── lib/
    └── other.jar  ← 无法被 ClassLoader 加载

Spring Boot JAR（可用）：
myapp.jar
└── BOOT-INF/lib/
    └── other.jar  ← 通过自定义 ClassLoader 加载
```

## 易错点/踩坑

- ❌ 混淆 Main-Class 和 Start-Class
- ❌ 打包成普通 JAR 无法执行
- ❌ 依赖的类找不到（未正确打包）

## 代码示例

### pom.xml 配置

```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <mainClass>com.example.Application</mainClass>  <!-- 可选 -->
    </configuration>
</plugin>
```

### 自定义 Main-Class

```properties
# MANIFEST.MF
Main-Class: com.example.CustomLauncher
Start-Class: com.example.Application
```

### 分层打包（2.3+）

```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <layers>
            <enabled>true</enabled>
        </layers>
    </configuration>
</plugin>
```

## 关联知识点

- spring-boot-maven-plugin 插件
- 打包与部署
- Docker 镜像构建
- Spring Boot Loader
