# Spring Boot Banner 自定义

## Q1：如何关闭 Spring Boot 的 Banner？

**A**：有三种方式：

**方式一：application.yml**
```yaml
spring:
  main:
    banner-mode: off
```

**方式二：代码**
```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication app = new SpringApplication(Application.class);
        app.setBannerMode(Banner.Mode.OFF);
        app.run(args);
    }
}
```

**方式三：命令行参数**
```bash
java -jar app.jar --spring.main.banner-mode=off
```

---

## Q2：Banner 支持哪些格式？

**A**：

| 格式 | 支持 | 说明 |
|------|------|------|
| `.txt` | ✅ | 文本格式，最常用 |
| `.gif/.jpg/.png` | ✅ | 图片格式（2.2+） |
| `.html` | ✅ | HTML 格式（2.4+） |

**图片 Banner 注意事项**：
```yaml
spring:
  banner:
    image:
      location: classpath:banner.png
      width: 100        # 字符宽度
      height: 50        # 字符高度
      margin: 1         # 左边距
```

---

## Q3：banner.txt 中可以使用哪些变量？

**A**：

| 变量 | 说明 | 输出示例 |
|------|------|----------|
| `${application.version}` | 应用版本 | v1.0.0 |
| `${application.formatted-version}` | 格式化版本 | v1.0.0 |
| `${spring-boot.version}` | Spring Boot 版本 | 3.2.0 |
| `${spring-boot.formatted-version}` | 格式化 Boot 版本 | 3.2.0 |
| `${title}` | 应用标题 | My Application |
| `${application.title}` | 应用标题 | My Application |

**示例**：
```
==========================================
   ${application.title}
   Version: ${application.version}
   Spring Boot: ${spring-boot.version}
==========================================
```

---

## Q4：如何生成一个 ASCII Banner？

**A**：可以使用在线工具生成：

1. **Text to ASCII Art Generator**: https://patorjk.com/software/taag/
2. **ASCII Banner Generator**: https://devops.datenkollektiv.de/banner/index.html

**推荐字体**：
- `Standard` - 通用
- `Shadow` - 带阴影
- `Banner` - 大字

---

## Q5：如何自定义 Banner 类？

**A**：实现 `Banner` 接口：

```java
public class CustomBanner implements Banner {
    
    @Override
    public void printBanner(Environment env, Class<?> sourceClass, PrintStream out) {
        // 获取版本信息
        String version = env.getProperty("spring.boot.version", "Unknown");
        String appName = env.getProperty("spring.application.name", "app");
        
        // 自定义输出
        out.println("╔════════════════════════════════════╗");
        out.println("║        " + appName + "               ║");
        out.println("║        Version: " + version + "            ║");
        out.println("╚════════════════════════════════════╝");
    }
}
```

**注册使用**：
```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication app = new SpringApplication(Application.class);
        app.setBanner(new CustomBanner());
        app.run(args);
    }
}
```

---

## Q6：Banner 图片支持动画吗？

**A**：**支持**！

**方式一：GIF 动画**
```
把 banner.gif 放到 classpath 下即可
Spring Boot 会自动播放动画
```

**方式二：HTML Banner**
```html
<!-- src/main/resources/banner.html -->
<!DOCTYPE html>
<html>
<body>
<h1 style="color: green;">My Spring Boot App</h1>
<p>Version: ${spring-boot.version}</p>
</body>
</html>
```

**配置**：
```yaml
spring:
  banner:
    image:
      location: classpath:banner.gif
    html:
      location: classpath:banner.html  # 2.4+
```

---

## Q7：Banner 打印顺序是怎样的？

**A**：`SpringApplication.run()` 中的顺序：

```
1. SpringApplication 初始化
        ↓
2. Banner 加载（从 classpath 读取）
        ↓
3. Banner 打印到 Console
        ↓
4. Banner 打印到 Log（如果 mode=console,log）
        ↓
5. 应用启动日志开始输出
```

**启动日志示例**（带 Banner）：
```
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::               (v3.2.0)

2026-04-28 10:30:00.123  INFO 12345 --- [           main] c.e.demo.DemoApplication     : Starting DemoApplication...
```
