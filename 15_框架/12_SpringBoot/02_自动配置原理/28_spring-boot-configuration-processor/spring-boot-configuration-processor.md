# spring-boot-configuration-processor

## 先说结论

- `spring-boot-configuration-processor` 是 **Maven/Gradle 注解处理器**
- 自动生成 `spring-configuration-metadata.json`，为 IDE 提供**配置属性提示**

## 深度解析

### 核心概念

```
工作流程：
1. 编译时扫描 @ConfigurationProperties 注解
2. 提取配置元信息（prefix、字段、默认值、描述）
3. 生成 META-INF/spring-configuration-metadata.json
4. IDE 读取元数据，提供代码提示
```

### Maven 配置

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-configuration-processor</artifactId>
    <optional>true</optional>
</dependency>
```

### 生成的元数据示例

```json
{
  "groups": [{
    "name": "myapp.feature",
    "type": "com.example.MyProperties",
    "sourceType": "com.example.MyProperties"
  }],
  "properties": [{
    "name": "myapp.feature.enabled",
    "type": "java.lang.Boolean",
    "sourceType": "com.example.MyProperties",
    "defaultValue": true
  }],
  "hints": [{
    "name": "myapp.feature.mode",
    "values": [
      {"value": "simple"}, {"value": "advanced"}, {"value": "expert"}
    ]
  }]
}
```

### IDE 提示效果

```
spring:
  myapp:
    feature:
      enabled: true   ← 自动提示 + 默认值显示
      mode: simple    ← 自动提示可枚举值
```

## 易错点/踩坑

- ❌ **忘记添加为 optional**：会打包到生产依赖
- ❌ **修改配置不重新编译**：元数据不会自动更新
- ❌ **私有字段无法识别**：getter/setter 才会生成元数据

## 代码示例

```java
@ConfigurationProperties(prefix = "myapp.feature")
public class MyProperties {
    
    private boolean enabled = true;
    
    private Mode mode = Mode.SIMPLE;
    
    public enum Mode { SIMPLE, ADVANCED, EXPERT }
    
    // getter/setter 必须有
    public boolean isEnabled() { return enabled; }
    public void setEnabled(boolean enabled) { this.enabled = enabled; }
    
    public Mode getMode() { return mode; }
    public void setMode(Mode mode) { this.mode = mode; }
}
```

## 关联知识点

- `@ConfigurationProperties` 配置绑定
- `@EnableConfigurationProperties` 启用配置
- Relaxed 属性名绑定
