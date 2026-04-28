# MultipartFile文件上传 - QA

## Q1：如何限制上传文件类型？

**A**：

```java
@PostMapping("/upload")
public Result upload(@RequestParam("file") MultipartFile file) {
    String contentType = file.getContentType();
    if (!contentType.startsWith("image/")) {
        return Result.error("只支持图片上传");
    }
    return Result.success(fileService.save(file));
}
```

---

## Q2：文件名中文乱码怎么办？

**A**：

```java
String filename = file.getOriginalFilename();
// 或使用编码转换
String name = new String(filename.getBytes(StandardCharsets.ISO_8859_1), StandardCharsets.UTF_8);
```

---

## Q3：文件太大怎么办？

**A**：配置文件大小限制。

```yaml
spring:
  servlet:
    multipart:
      max-file-size: 5MB
      max-request-size: 50MB
```

---

## Q4：如何获取上传进度？

**A**：Servlet 3.0+ 不支持进度监听，建议使用前端库（如 fine-uploader）或自定义 Filter。

---

## Q5：文件上传到云存储？

**A**：

```java
public String uploadToOSS(MultipartFile file) {
    // 使用 OSS SDK
    OSS ossClient = new OSSClient(endpoint, accessKeyId, accessKeySecret);
    ossClient.putObject(bucketName, key, file.getInputStream());
    return ossClient.getUrl(bucketName, key).toString();
}
```
