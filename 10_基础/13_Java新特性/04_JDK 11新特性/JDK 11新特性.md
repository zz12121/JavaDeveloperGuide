---
title: JDK 11新特性
tags:
  - Java/新特性
  - 原理型
module: 13_Java新特性
created: 2026-04-18
---

# String 改进

## 核心结论

JDK 11 对 `String` 类新增了多个实用方法：`isBlank()`、`lines()`、`strip()`/`stripLeading()`/`stripTrailing()`、`repeat()`、`formatted()`（JDK 15）等，简化了常见的字符串处理操作。JDK 12 新增了 `indent()`、`transform()` 等方法。这些方法大多是对已有常见操作的标准化封装。

---

## 深度解析

### 1. isBlank() — 判断空白（JDK 11）

```java
"  ".isBlank();       // true（空格也是空白）
"\t\n".isBlank();     // true（制表符、换行符）
"".isBlank();         // true
"hello".isBlank();    // false

// 对比 isEmpty()
"  ".isEmpty();       // false（有内容，只是空格）
"".isEmpty();         // true
```

### 2. strip() 系列 — 去除空白（JDK 11）

```java
String s = "  hello  ";

s.strip();            // "hello"（去除首尾 Unicode 空白，包括 \u2003 等）
s.stripLeading();     // "hello  "（去除开头空白）
s.stripTrailing();    // "  hello"（去除末尾空白）

// 与 trim() 的区别
String s2 = "\u2003hello\u2003";  // \u2003 是全角空格（Em Space）
s2.trim();            // "\u2003hello\u2003"（trim 不识别 Unicode 空白）
s2.strip();           // "hello"（strip 识别所有 Unicode 空白字符）
```

> `strip()` 基于 `Character.isWhitespace()`，`trim()` 只去除 `char <= ' '` 的字符。

### 3. lines() — 按行分割（JDK 11）

```java
String multiline = "line1\nline2\r\nline3";

List<String> lines = multiline.lines().collect(Collectors.toList());
// ["line1", "line2", "line3"]

// 对比 split()
multiline.split("\\R");  // 需要正则，lines() 更简洁
```

### 4. repeat() — 重复字符串（JDK 11）

```java
"abc".repeat(3);   // "abcabcabc"
"*".repeat(5);     // "*****"
" ".repeat(4);     // "    "
```

### 5. formatted() — 格式化（JDK 15）

```java
// JDK 15 之前
String.format("Hello, %s!", name);

// JDK 15 之后
"Hello, %s!".formatted(name);
```

### 6. 其他实用方法

```java
// transform() — 链式转换（JDK 12）
"hello".transform(s -> s.toUpperCase())
       .transform(s -> ">> " + s + " <<");
// ">> HELLO <<"

// indent() — 缩进（JDK 12）
"line1\nline2".indent(4);
// "    line1\n    line2\n"

// String.compareToIgnoreCase 的常量形式
String s = "hello";
s.equalsIgnoreCase("HELLO");  // true

// describeConstable() — 返回 Optional<String>（JDK 12，配合 Constable 接口）
"hello".describeConstable();  // Optional[hello]
```

### 7. JDK 8~21 String 方法速查

| JDK | 方法 | 说明 |
|-----|------|------|
| 8 | `join()` | 静态方法，拼接字符串 |
| 11 | `isBlank()` | 判断是否为空白 |
| 11 | `strip()` / `stripLeading()` / `stripTrailing()` | 去除 Unicode 空白 |
| 11 | `lines()` | 按行分割为 Stream |
| 11 | `repeat(int)` | 重复 n 次 |
| 12 | `indent(int)` | 缩进调整 |
| 12 | `transform(Function)` | 链式转换 |
| 15 | `formatted(Object...)` | 格式化字符串 |
| 15 | `stripIndent()` | 去除缩进 |

---

## 代码示例

```java
// 实际场景：处理用户输入
String input = "  \n  hello\n  \n  world\n  ";

// JDK 11+
input.lines()
     .map(String::strip)
     .filter(s -> !s.isBlank())
     .collect(Collectors.joining(", "));
// "hello, world"

// 生成 CSV 行
String csv = String.join(",", List.of("name", "age", "city"));
// "name,age,city"
```

---

---

# 全新 HTTP Client API

## 核心结论

JDK 11 引入了标准化的 `java.net.http` 包（JEP 321），提供了**同步**和**响应式**两种 HTTP 通信方式，替代了 Apache HttpClient、OkHttp 等第三方库。底层基于 NIO（Selector），支持 HTTP/1.1 和 **HTTP/2**，支持 WebSocket，是 JDK 历史上最完整的 HTTP 客户端。

---

## 深度解析

### 1. 为什么需要新的 HTTP Client？

| 对比项 | `HttpURLConnection`（JDK 1.0）| 第三方库（OkHttp/Apache）| JDK 11+ `HttpClient` |
|--------|---------------------------|------------------------|---------------------|
| HTTP 版本 | 仅 HTTP/1.1 | HTTP/1.1 / HTTP/2 | HTTP/1.1 / HTTP/2 |
| API 设计 | 繁琐，基于流 | 简洁，链式 | 链式，流式 |
| 异步支持 | 无 | 支持 | 支持（CompletableFuture）|
| WebSocket | 无 | 支持 | 支持 |
| JDK 内置 | ✅ | ❌ 需额外依赖 | ✅ |
| 响应式 | 不支持 | 支持 | 支持（BodyPublishers/BodySubscribers）|

### 2. 基本用法：同步请求

```java
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;

// 创建 HttpClient
HttpClient client = HttpClient.newHttpClient();

// 构建请求
HttpRequest request = HttpRequest.newBuilder()
    .uri(URI.create("https://api.example.com/users/1"))
    .header("Content-Type", "application/json")
    .header("Authorization", "Bearer token123")
    .timeout(Duration.ofSeconds(10))
    .GET()  // 或 .POST(HttpRequest.BodyPublishers.ofString(json))
    .build();

// 同步发送
HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());

// 处理响应
System.out.println("状态码：" + response.statusCode());
System.out.println("响应体：" + response.body());
System.out.println("响应头：" + response.headers());
```

### 3. 异步请求

```java
// 异步发送（返回 CompletableFuture）
HttpResponse<String> response = client.sendAsync(request, HttpResponse.BodyHandlers.ofString())
    .thenApply(HttpResponse::body)
    .thenAccept(System.out::println)
    .exceptionally(ex -> {
        System.err.println("请求失败：" + ex.getMessage());
        return null;
    });

// 等待完成
response.join();
```

### 4. POST 请求与请求体

```java
String jsonBody = """
    {
        "name": "张三",
        "age": 25
    }
    """;

HttpRequest postRequest = HttpRequest.newBuilder()
    .uri(URI.create("https://api.example.com/users"))
    .header("Content-Type", "application/json")
    .POST(HttpRequest.BodyPublishers.ofString(jsonBody))
    .build();

// 文件作为请求体
HttpRequest fileRequest = HttpRequest.newBuilder()
    .uri(URI.create("https://api.example.com/upload"))
    .header("Content-Type", "application/octet-stream")
    .POST(HttpRequest.BodyPublishers.ofFile(Path.of("file.pdf")))
    .build();
```

### 5. HTTP/2 与连接复用

```java
// 默认启用 HTTP/2（自动协商，降级到 HTTP/1.1 如果服务器不支持）
HttpClient client = HttpClient.newBuilder()
    .version(HttpClient.Version.HTTP_2)  // 显式指定版本
    .connectTimeout(Duration.ofSeconds(5))
    .executor(Executors.newFixedThreadPool(10))  // 自定义线程池
    .build();

// HTTP/2 多路复用：一个 TCP 连接上并行多个请求
// 比 HTTP/1.1 每个请求一个连接效率高得多
```

### 6. Cookie 管理与代理

```java
HttpClient client = HttpClient.newBuilder()
    .cookieHandler(new CookieManager())  // 自动管理 Cookie
    .proxy(ProxySelector.of(new InetSocketAddress("proxy.example.com", 8080)))
    .sslContext(SSLContext.getDefault())  // 自定义 SSL 上下文
    .authenticator(new Authenticator() {
        @Override
        protected PasswordAuthentication getPasswordAuthentication() {
            return new PasswordAuthentication("user", "password".toCharArray());
        }
    })
    .build();
```

### 7. WebSocket 客户端

```java
HttpClient wsClient = HttpClient.newHttpClient();

HttpRequest wsRequest = HttpRequest.newBuilder()
    .uri(URI.create("wss://echo.websocket.org"))
    .build();

WebSocket ws = wsClient.newWebSocketBuilder()
    .buildAsync(wsRequest, new WebSocket.Listener() {
        @Override
        public void onOpen(WebSocket webSocket) {
            System.out.println("连接建立");
            webSocket.sendText("Hello Server", true);
        }

        @Override
        public void onText(WebSocket webSocket, CharSequence data, boolean last) {
            System.out.println("收到消息：" + data);
        }

        @Override
        public void onClose(WebSocket webSocket, int statusCode, String reason) {
            System.out.println("连接关闭：" + statusCode + " " + reason);
        }
    })
    .join();
```

### 8. 响应式处理（BodySubscribers）

```java
// 流式处理大文件（不占用大量内存）
Path filePath = Path.of("download.zip");
HttpResponse<Path> response = client.sendAsync(request,
    HttpResponse.BodyHandlers.ofFile(filePath))
    .join();

// 将响应体解析为 JSON 节点
HttpResponse<JsonNode> jsonResponse = client.sendAsync(request,
    HttpResponse.BodyHandlers.ofJson())
    .join();

// 自定义响应处理
HttpResponse<String> response = client.sendAsync(request,
    HttpResponse.BodyHandlers.ofString())
    .join();
```

---

## 代码示例：实际应用场景

```java
// 场景：并发调用多个接口并汇总结果
public Map<String, Object> fetchUserProfile(Long userId) throws Exception {
    HttpClient client = HttpClient.newHttpClient();

    HttpRequest userReq = HttpRequest.newBuilder()
        .uri(URI.create("https://api.example.com/users/" + userId))
        .GET().build();

    HttpRequest orderReq = HttpRequest.newBuilder()
        .uri(URI.create("https://api.example.com/users/" + userId + "/orders"))
        .GET().build();

    // 并发请求
    HttpResponse<String> userRes = client.sendAsync(userReq, HttpResponse.BodyHandlers.ofString())
        .get(5, TimeUnit.SECONDS);
    HttpResponse<String> orderRes = client.sendAsync(orderReq, HttpResponse.BodyHandlers.ofString())
        .get(5, TimeUnit.SECONDS);

    // 合并结果
    Map<String, Object> result = new HashMap<>();
    result.put("user", objectMapper.readTree(userRes.body()));
    result.put("orders", objectMapper.readTree(orderRes.body()));
    return result;
}
```

---

## 关联知识点



