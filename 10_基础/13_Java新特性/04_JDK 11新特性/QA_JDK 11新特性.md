---
title: JDK 11新特性
tags:
  - Java/新特性
  - 问答
  - 原理型
module: 13_Java新特性
created: 2026-04-18
---

# String 改进

## Q1：strip() 和 trim() 有什么区别？

**A**：
- `trim()`：只去除 ASCII 码小于等于空格（`' '`，即 `\u0020`）的空白字符
- `strip()`：基于 `Character.isWhitespace()`，能识别所有 Unicode 空白字符（包括全角空格 `\u2003`、不间断空格 `\u00A0` 等）
```java
String s = "\u2003hello\u2003";
s.trim();   // "\u2003hello\u2003"（无法去除）
s.strip();  // "hello"（正常去除）
```

---

## Q2：isBlank() 和 isEmpty() 有什么区别？

**A**：
- `isEmpty()`：长度为 0 时返回 true（`str.length() == 0`）
- `isBlank()`：只包含空白字符或长度为 0 时返回 true
```java
"  ".isEmpty();    // false（有内容）
"  ".isBlank();   // true（只有空白）
"".isEmpty();      // true
"".isBlank();     // true
```

---

## Q3：lines() 和 split("\\R") 有什么区别？

**A**：功能相似，但 `lines()` 有优势：
- `lines()` 返回 `Stream<String>`，可以直接链式处理
- `lines()` 不会返回末尾的空字符串（trailing empty line）
- `split("\\R")` 返回数组，且可能在末尾包含空字符串
```java
"a\nb\n".lines().collect(Collectors.toList());  // ["a", "b"]
"a\nb\n".split("\\R");                          // ["a", "b"]
```

---

## Q4：repeat() 有什么实际用途？

**A**：常见用途包括生成分隔线、缩进、填充等：
```java
"-".repeat(50);           // 生成 50 个减号的分隔线
" ".repeat(4);            // 生成 4 个空格的缩进
"0".repeat(8 - hash.length()); // 哈希值前补零
```

---

# 全新 HTTP Client API

## Q5：JDK 11 的 HttpClient 和 HttpURLConnection 有什么区别？

**A**：

| 对比项 | `HttpURLConnection`（JDK 1.0）| `HttpClient`（JDK 11）|
|--------|---------------------------|----------------------|
| API 复杂度 | 繁琐（流式 API，错误处理复杂）| 链式 API，简洁直观 |
| HTTP/2 支持 | ❌ 不支持 | ✅ 支持（自动协商）|
| 异步支持 | ❌ 无 | ✅ CompletableFuture |
| WebSocket | ❌ 不支持 | ✅ 支持 |
| 连接复用 | ❌ 每次请求新建连接 | ✅ HTTP/2 多路复用 |
| 代理/Cookie | 手动管理 | 内置支持 |
| 响应式流 | ❌ 不支持 | ✅ BodyPublishers/Subscribers |

```java
// HttpURLConnection：繁琐
HttpURLConnection conn = (HttpURLConnection) url.openConnection();
conn.setRequestMethod("GET");
conn.connect();
int code = conn.getResponseCode();  // ...

// HttpClient：简洁
HttpResponse<String> resp = HttpClient.newHttpClient()
    .send(HttpRequest.newBuilder(uri).GET().build(),
          HttpResponse.BodyHandlers.ofString());
```

---

## Q6：HttpClient 支持哪些 HTTP 版本？如何选择？

**A**：

`HttpClient` 支持两种版本：

- `HTTP_1_1`：默认，回退版本，所有服务器都支持
- `HTTP_2`：JDK 11+ 默认首选，支持多路复用（一个 TCP 连接并行多个请求）

```java
// 默认行为（HTTP/2，服务器不支持时自动降级）
HttpClient client = HttpClient.newHttpClient();

// 强制指定版本
HttpClient client = HttpClient.newBuilder()
    .version(HttpClient.Version.HTTP_2)
    .build();

// 强制使用 HTTP/1.1
HttpClient client = HttpClient.newBuilder()
    .version(HttpClient.Version.HTTP_1_1)
    .build();
```

**选择建议**：一般用默认即可。HTTP/2 适合高并发场景（减少 TCP 连接数），HTTP/1.1 适合需要兼容老旧服务器的场景。

---

## Q7：HttpClient 如何处理异步请求和超时？

**A**：

```java
HttpClient client = HttpClient.newHttpClient();

HttpRequest request = HttpRequest.newBuilder()
    .uri(URI.create("https://api.example.com/data"))
    .timeout(Duration.ofSeconds(5))  // 请求超时
    .GET()
    .build();

// 异步请求
CompletableFuture<HttpResponse<String>> future =
    client.sendAsync(request, HttpResponse.BodyHandlers.ofString());

// 设置全局超时
future.orTimeout(5, TimeUnit.SECONDS)
    .thenAccept(resp -> System.out.println(resp.body()))
    .exceptionally(ex -> {
        if (ex.getCause() instanceof TimeoutException) {
            System.out.println("请求超时");
        }
        return null;
    });
```

---

## Q8：HttpClient 支持 WebSocket 吗？如何使用？

**A**：**支持**，`JDK 11+` 提供了完整的 WebSocket 客户端：

```java
HttpClient client = HttpClient.newHttpClient();
WebSocket ws = client.newWebSocketBuilder()
    .buildAsync(URI.create("wss://echo.websocket.org"), new WebSocket.Listener() {
        @Override
        public void onOpen(WebSocket webSocket) {
            webSocket.sendText("Hello", true);  // 发送消息
        }

        @Override
        public void onText(WebSocket webSocket, CharSequence data, boolean last) {
            System.out.println("收到: " + data);
            webSocket.sendClose(WebSocket.NORMAL_CLOSURE, "done");
        }
    }).join();  // 阻塞等待连接建立
```

> **注意**：JDK 11 的 WebSocket 是**客户端实现**，不包含服务端。如需服务端，使用 Spring WebSocket 或 Netty。

**A**：常见用途包括生成分隔线、缩进、填充等：
```java
"-".repeat(50);           // 生成 50 个减号的分隔线
" ".repeat(4);            // 生成 4 个空格的缩进
"0".repeat(8 - hash.length()); // 哈希值前补零
```
