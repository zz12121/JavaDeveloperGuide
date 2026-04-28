# HttpMessageConverters自动配置

## 先说结论

`HttpMessageConverter` 负责请求体/响应体的数据转换。SpringBoot 自动配置了 JSON、XML、文本等常用转换器，**开箱即用**。

## 深度解析

### 核心概念

| 类 | 作用 |
|---|------|
| HttpMessageConverter | 消息转换接口 |
| MappingJackson2HttpMessageConverter | JSON 转换 |
| Jaxb2RootElementHttpMessageConverter | XML 转换 |
| StringHttpMessageConverter | 文本转换 |
| ByteArrayHttpMessageConverter | 字节数组转换 |

### 自动配置的转换器

| 转换器 | 支持类型 | 优先级 |
|--------|----------|--------|
| ByteArrayHttpMessageConverter | `byte[]` | 1 |
| StringHttpMessageConverter | `String` | 2 |
| ResourceHttpMessageConverter | `Resource` | 3 |
| MappingJackson2HttpMessageConverter | JSON | 4 |
| Jaxb2RootElementHttpMessageConverter | XML | 5 |

### 请求/响应处理流程

```java
// 请求处理
@RequestBody User user
        ↓
HttpMessageConverter.read()
        ↓
String → User 对象转换

// 响应处理
@ResponseBody User user
        ↓
HttpMessageConverter.write()
        ↓
User → JSON 字符串
```

### 自定义转换器

```java
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {
    
    @Override
    public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
        // 完全替换默认转换器
        converters.add(new Jackson2ObjectMapperBuilder().createXmlMapper(true).build());
    }
    
    @Override
    public void extendMessageConverters(List<HttpMessageConverter<?>> converters) {
        // 在默认转换器基础上追加
        converters.add(0, new CustomHttpMessageConverter());
    }
}
```

## 易错点/踩坑

- ❌ `configureMessageConverters` 会**替换**默认转换器，导致 JSON 转换失效
- ❌ `extendMessageConverters` 是**追加**到末尾，可能导致优先级问题
- ❌ 返回 XML 时忘记添加 JAXB2 依赖

## 代码示例

### 全局配置 JSON 日期格式

```java
@Configuration
public class JacksonConfig {
    @Bean
    public ObjectMapper objectMapper() {
        ObjectMapper mapper = new ObjectMapper();
        mapper.dateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        mapper.serializationInclusion(JsonInclude.Include.NON_NULL);
        return mapper;
    }
}
```

### 自定义 FastJson 转换器

```xml
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>fastjson</artifactId>
    <version>2.0.43</version>
</dependency>
```

```java
@Configuration
public class FastJsonConfig {
    @Bean
    public HttpMessageConverters fastJsonConverter() {
        FastJsonHttpMessageConverter converter = new FastJsonHttpMessageConverter();
        FastJsonConfig config = new FastJsonConfig();
        config.setDateFormat("yyyy-MM-dd HH:mm:ss");
        converter.setFastJsonConfig(config);
        return new HttpMessageConverters(converter);
    }
}
```

## 图解/流程

```
请求 → HttpMessageConverter.read()
        ↓
┌─────────────────────────────────────┐
│ Content-Type: application/json     │
│     → MappingJackson2HttpMessageConverter │
├─────────────────────────────────────┤
│ Content-Type: application/xml      │
│     → Jaxb2RootElementHttpMessageConverter │
├─────────────────────────────────────┤
│ Content-Type: text/plain           │
│     → StringHttpMessageConverter   │
└─────────────────────────────────────┘
        ↓
Java 对象
```

## 关联知识点

- `11_@RequestBody请求体`：使用 @RequestBody 触发消息转换
- `12_@ResponseBody响应体`：使用 @ResponseBody 触发消息转换
