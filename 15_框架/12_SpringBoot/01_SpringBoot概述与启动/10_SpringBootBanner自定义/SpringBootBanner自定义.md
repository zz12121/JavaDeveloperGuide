# Spring Boot Banner 自定义

## 先说结论

Banner 是 Spring Boot 启动时打印的 ASCII 艺术字，可以通过文本、图片或代码方式自定义，或完全关闭。

## 深度解析

### Banner 打印流程

```
┌─────────────────────────────────────────────────────────────┐
│                    Banner 打印流程                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   SpringApplication.run()                                   │
│         │                                                  │
│         ├── Banner 准备                                    │
│         │    │                                            │
│         │    └── 从以下位置加载 Banner：                   │
│         │         1. classpath:banner.gif/jpg/png         │
│         │         2. classpath:banner.txt                 │
│         │         3. spring.banner.* 配置                  │
│         │                                                  │
│         └── Banner 打印                                    │
│              │                                             │
│              └── Console / Log / OFF                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Banner 位置优先级

| 优先级 | 位置 | 说明 |
|--------|------|------|
| 1 | `classpath:banner.gif/jpg/png` | 图片 Banner |
| 2 | `classpath:banner.txt` | 文本 Banner |
| 3 | `spring.banner.*` 配置 | application.yml 中的配置 |

### Banner 配置项

```yaml
spring:
  banner:
    charset: UTF-8              # 编码
    image:
      location: classpath:xxx.png  # 图片位置
      width: 80                # 宽度
      height: 40               # 高度
    location: classpath:banner.txt  # 文本位置
```

### Banner 模式

| 模式 | 说明 |
|------|------|
| `CONSOLE` | 打印到控制台（默认） |
| `LOG` | 打印到日志 |
| `OFF` | 关闭 Banner |

## 易错点/踩坑

- ❌ Banner 图片格式不支持
- ❌ Banner 编码乱码
- ❌ 关闭 Banner 方式错误

## 代码示例

### 方式一：banner.txt 文件

在 `src/main/resources/` 下创建 `banner.txt`：

```
 _____      _            _               _ 
/ ____|    | |          | |             | |
| | ___  _ | | ___  _ __| | ___   __ _  | |
| |/ _ \| | |/ _ \| '__| |/ _ \ / _` | | |
| | (_) | | | (_) | |  | | (_) | (_| | |_|
 \_____/|_|_|\___/|_|  |_|\___/ \__, | (_)
                                  __/ |    
                                 |___/     

:: Spring Boot ::        (v${spring-boot.version})
:: Application ::        ${spring.application.name}
```

### 方式二：自定义 Banner 类

```java
public class MyBanner implements Banner {
    
    @Override
    public void printBanner(Environment env, Class<?> sourceClass, PrintStream out) {
        String banner = "===========================\n" +
                        "   My Custom Application  \n" +
                        "   Version: " + env.getProperty("version", "1.0.0") + "\n" +
                        "===========================";
        out.println(banner);
    }
}
```

### 方式三：关闭 Banner

```java
// 方式一：代码
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication app = new SpringApplication(Application.class);
        app.setBannerMode(Banner.Mode.OFF);
        app.run(args);
    }
}

// 方式二：application.yml
spring:
  main:
    banner-mode: off

// 方式三：命令行
java -jar app.jar --spring.main.banner-mode=off
```

### 可用变量

| 变量 | 说明 | 示例 |
|------|------|------|
| `${application.version}` | 应用版本 | v1.0.0 |
| `${application.formatted-version}` | 格式化版本 | v1.0.0 |
| `${spring-boot.version}` | Spring Boot 版本 | 3.2.0 |
| `${spring-boot.formatted-version}` | 格式化 Boot 版本 | 3.2.0 |
| `${title}` | 应用标题 | Application |
| `${application.title}` | 同上 | Application |

## 关联知识点

- SpringApplication 静态方法
- `SpringApplicationBuilder` 链式构建
- application.yml 配置
