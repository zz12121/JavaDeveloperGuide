# Servlet API原生对象注入

## 先说结论

Spring MVC 允许直接将 Servlet API 对象作为方法参数注入，包括 `HttpServletRequest`、`HttpServletResponse`、`HttpSession` 等。

## 深度解析

### 支持的 Servlet 参数

| 参数类型 | 说明 |
|----------|------|
| HttpServletRequest | HTTP 请求对象 |
| HttpServletResponse | HTTP 响应对象 |
| HttpSession | HTTP 会话对象 |
| Principal | 当前用户主体 |
| Locale | 本地化信息 |
| InputStream/Reader | 请求输入流 |
| OutputStream/Writer | 响应输出流 |

### 基本使用

```java
@GetMapping("/request")
public String handleRequest(
    HttpServletRequest request,
    HttpServletResponse response,
    HttpSession session
) {
    String method = request.getMethod();
    String uri = request.getRequestURI();
    session.setAttribute("lastVisit", new Date());
    return "Request: " + method + " " + uri;
}
```

### 获取请求信息

```java
@GetMapping("/info")
public Map<String, Object> getInfo(HttpServletRequest request) {
    Map<String, Object> info = new HashMap<>();
    info.put("method", request.getMethod());
    info.put("uri", request.getRequestURI());
    info.put("remoteAddr", request.getRemoteAddr());
    info.put("headers", Collections.list(request.getHeaderNames())
            .stream()
            .collect(Collectors.toMap(h -> h, request::getHeader)));
    return info;
}
```

### 操作响应

```java
@GetMapping("/download")
public void download(HttpServletResponse response) throws IOException {
    response.setContentType("application/octet-stream");
    response.setHeader("Content-Disposition", "attachment;filename=file.txt");
    
    try (OutputStream os = response.getOutputStream()) {
        os.write("Hello World".getBytes());
        os.flush();
    }
}
```

## 易错点/踩坑

- ❌ 在 @ResponseBody 方法中使用 HttpServletResponse 获取输出流 → 与消息转换器冲突
- ❌ 直接操作 response.getWriter() → 可能与视图解析器冲突
- ❌ Session 为 null → 浏览器禁用 Cookie 或请求未携带 JSESSIONID

## 代码示例

### 文件上传（原始方式）

```java
@PostMapping("/upload")
public String upload(HttpServletRequest request) throws Exception {
    DiskFileItemFactory factory = new DiskFileItemFactory();
    ServletFileUpload upload = new ServletFileUpload(factory);
    List<FileItem> items = upload.parseRequest(request);
    
    for (FileItem item : items) {
        if (!item.isFormField()) {
            String fileName = item.getName();
            item.write(new File("/tmp/" + fileName));
        }
    }
    return "redirect:/success";
}
```

### 重定向

```java
@GetMapping("/redirect")
public void redirect(HttpServletResponse response) throws IOException {
    response.sendRedirect("/target");
}

// 或使用 ResponseEntity
@GetMapping("/redirect")
public ResponseEntity<Void> redirect() {
    return ResponseEntity.status(HttpStatus.FOUND)
            .header("Location", "/target")
            .build();
}
```

## 关联知识点

- `07_@RequestMapping请求映射`：请求映射基础
- `30_MultipartFile文件上传`：推荐的文件上传方式
