# ContentNegotiation内容协商

## 先说结论

内容协商允许同一接口返回不同格式的数据（JSON/XML/HTML），Spring 根据请求头 `Accept` 或参数自动选择合适的格式。

## 深度解析

### 核心概念

| 概念 | 说明 |
|------|------|
| ContentNegotiationStrategy | 内容协商策略接口 |
| AcceptHeaderContentNegotiationStrategy | 基于 Accept 头协商 |
| ParameterContentNegotiationStrategy | 基于请求参数协商 |
| PathExtensionContentNegotiationStrategy | 基于 URL 后缀协商 |

### 协商流程

```
请求 → Accept: application/json
        ↓
ContentNegotiationManager
        ↓
查找匹配的 HttpMessageConverter
        ↓
使用 MappingJackson2HttpMessageConverter
        ↓
返回 JSON
```

### 配置多格式支持

```java
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {
    
    @Override
    public void configureContentNegotiation(ContentNegotiationConfigurer configurer) {
        configurer
            .favorParameter(true)           // 开启参数协商 ?format=json
            .parameterName("format")        // 参数名
            .defaultContentType(MediaType.APPLICATION_JSON)  // 默认格式
            .mediaType("json", MediaType.APPLICATION_JSON)
            .mediaType("xml", MediaType.APPLICATION_XML)
            .mediaType("html", MediaType.TEXT_HTML);
    }
}
```

### URL 后缀协商

```yaml
spring:
  mvc:
    contentnegotiation:
      favor-path-extension: true   # 开启后缀协商
      media-types:                 # 自定义后缀映射
        yml: application/x-yaml
```

```bash
# 示例
GET /user/1.json   → 返回 JSON
GET /user/1.xml    → 返回 XML
GET /user/1.html   → 返回 HTML
```

## 易错点/踩坑

- ❌ 参数协商和路径后缀同时开启 → 路径后缀优先，可能冲突
- ❌ 返回 XML 但没有 JAXB 依赖 → 报错 `Could not find MessageConverter`
- ❌ 浏览器访问返回 HTML，接口调用返回 JSON → 浏览器默认 Accept 为 `text/html`

## 代码示例

### 接口同时支持 JSON 和 XML

```java
@RestController
@RequestMapping("/api/user")
public class UserController {
    
    @GetMapping("/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.getById(id);
    }
}
```

```bash
# 请求 JSON
curl -H "Accept: application/json" http://localhost:8080/api/user/1

# 请求 XML
curl -H "Accept: application/xml" http://localhost:8080/api/user/1
```

### 浏览器访问返回 HTML，接口返回 JSON

```java
// 浏览器直接访问，返回 HTML 页面
@GetMapping(value = "/list", produces = "text/html")
public String listPage() {
    return "user/list";
}

// API 调用，返回 JSON
@GetMapping(value = "/list", produces = "application/json")
@ResponseBody
public List<User> listApi() {
    return userService.list();
}
```

## 图解/流程

```
┌─────────────────────────────────────────────────┐
│              ContentNegotiation                 │
├─────────────────────────────────────────────────┤
│  1. Accept Header                               │
│     Accept: application/json                   │
│           application/xml                       │
├─────────────────────────────────────────────────┤
│  2. URL Path Extension                          │
│     /user/1.json                                │
├─────────────────────────────────────────────────┤
│  3. Request Parameter                           │
│     /user/1?format=json                         │
├─────────────────────────────────────────────────┤
│  4. Fixed Content Type (produces)              │
│     @GetMapping(produces = "application/json")  │
└─────────────────────────────────────────────────┘
        ↓
匹配 HttpMessageConverter
        ↓
返回对应格式
```

## 关联知识点

- `05_HttpMessageConverters自动配置`：转换器是协商的基础
- `12_@ResponseBody响应体`：@ResponseBody 触发内容协商
