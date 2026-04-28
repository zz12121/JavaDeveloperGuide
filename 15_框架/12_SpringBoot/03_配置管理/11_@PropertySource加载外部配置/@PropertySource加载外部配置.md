# @PropertySource 加载外部配置

## 先说结论

`@PropertySource` 用于加载指定的 `.properties` 外部配置文件，适合需要加载非 application 配置文件或遗留配置的场景。

## 深度解析

### 基本用法

```java
@Configuration
@PropertySource("classpath:db.properties")
@PropertySource("file:./custom.properties")
public class AppConfig {
    // 加载 db.properties 和 custom.properties
}
```

### 配合 @Value 使用

```java
@Configuration
@PropertySource("classpath:app.properties")
public class AppConfig {
    @Value("${db.url}")
    private String dbUrl;
}
```

### 不支持 YAML

**注意**：`@PropertySource` 不支持 YAML 文件，只支持 `.properties`。

```java
// ❌ 不支持
@PropertySource("classpath:config.yml")

// ✅ 支持
@PropertySource("classpath:config.properties")
```

### YAML 转 properties

如果必须用 YAML，可以手动加载：

```java
@Configuration
public class YamlConfig {
    @Bean
    public PropertySourcesPlaceholderConfigurer properties() {
        YamlPropertiesFactoryBean yaml = new YamlPropertiesFactoryBean();
        yaml.setResources(new ClassPathResource("config.yml"));

        PropertySourcesPlaceholderConfigurer configurer = new PropertySourcesPlaceholderConfigurer();
        configurer.setProperties(yaml.getObject());
        return configurer;
    }
}
```

### 多文件加载

```java
@PropertySources({
    @PropertySource("classpath:db.properties"),
    @PropertySource(value = "file:./override.properties", ignoreResourceNotFound = true)
})
```

### ignoreResourceNotFound

```java
// 文件不存在时忽略（不报错）
@PropertySource(value = "file:./optional.properties", ignoreResourceNotFound = true)
```

## 易错点/踩坑

- ❌ `@PropertySource` 不支持 YAML — 只支持 .properties
- ❌ 文件路径写错 — 启动报错，建议加 `ignoreResourceNotFound = true`
- ❌ 加载顺序问题 — 后加载的同名属性会覆盖前面的
- ❌ 不支持 profile 变体 — 如 `db-dev.properties` 不生效

## 关联知识点

- [[12_@PropertySources多文件加载]] — 多个配置文件
- [[13_@ImportResource导入XML]] — 导入 XML 配置
- [[14_SpringBoot配置优先级]] — 加载顺序
