# 文件下载实现 - QA

## Q1：Content-Disposition 头的区别？

**A**：

| 值 | 行为 |
|---|------|
| `attachment` | 弹出下载框 |
| `inline` | 浏览器直接显示（如图片） |

```java
headers.add(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=" + filename);
headers.add(HttpHeaders.CONTENT_DISPOSITION, "inline; filename=" + filename);
```

---

## Q2：文件名中文怎么办？

**A**：

```java
String encodedFilename = URLEncoder.encode(filename, StandardCharsets.UTF_8);
headers.add(HttpHeaders.CONTENT_DISPOSITION, 
        "attachment; filename=\"" + encodedFilename + "\"; filename*=UTF-8''" + encodedFilename);
```

---

## Q3：大文件下载用什么方式？

**A**：`StreamingResponseBody` 避免内存溢出。

```java
@GetMapping("/download/{filename}")
public ResponseEntity<StreamingResponseBody> download(
        @PathVariable String filename, HttpServletResponse response) {
    
    StreamingResponseBody stream = outputStream -> {
        Files.copy(file.toPath(), outputStream);
    };
    
    return ResponseEntity.ok()
            .contentType(MediaType.APPLICATION_OCTET_STREAM)
            .body(stream);
}
```

---

## Q4：如何限制下载权限？

**A**：

```java
@GetMapping("/download/{filename}")
public ResponseEntity<Resource> download(
        @PathVariable String filename,
        @AuthenticationPrincipal UserDetails user) {
    
    if (!user.hasPermission(filename)) {
        return ResponseEntity.status(HttpStatus.FORBIDDEN).build();
    }
    // ...
}
```
