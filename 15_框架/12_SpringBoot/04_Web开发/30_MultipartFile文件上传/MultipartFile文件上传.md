# MultipartFile文件上传

## 先说结论

`MultipartFile` 是 Spring 封装文件上传的核心类，提供文件读取、保存、元数据获取等功能。

## 深度解析

### MultipartFile 接口方法

| 方法 | 说明 |
|------|------|
| `getOriginalFilename()` | 获取原始文件名 |
| `getContentType()` | 获取文件类型 |
| `getSize()` | 获取文件大小 |
| `getBytes()` | 获取字节数组 |
| `getInputStream()` | 获取输入流 |
| `transferTo(File)` | 保存到文件 |

### 基本使用

```java
@PostMapping("/upload")
public Result upload(@RequestParam("file") MultipartFile file) {
    if (file.isEmpty()) {
        return Result.error("文件不能为空");
    }
    
    String filename = file.getOriginalFilename();
    String suffix = filename.substring(filename.lastIndexOf("."));
    
    // 保存文件
    String newFilename = UUID.randomUUID() + suffix;
    File dest = new File("/uploads/" + newFilename);
    file.transferTo(dest);
    
    return Result.success("/uploads/" + newFilename);
}
```

### 多文件上传

```java
@PostMapping("/uploadMultiple")
public Result uploadMultiple(@RequestParam("files") MultipartFile[] files) {
    List<String> urls = new ArrayList<>();
    for (MultipartFile file : files) {
        if (!file.isEmpty()) {
            String url = fileService.save(file);
            urls.add(url);
        }
    }
    return Result.success(urls);
}
```

## 易错点/踩坑

- ❌ 文件名为空 → 检查 `enctype="multipart/form-data"`
- ❌ 中文文件名乱码 → 使用 `new String(name.getBytes(), "ISO-8859-1")`
- ❌ 上传目录不存在 → 提前创建或使用配置路径

## 代码示例

### 完整上传服务

```java
@Service
public class FileService {
    
    @Value("${upload.path:/uploads}")
    private String uploadPath;
    
    public String save(MultipartFile file) {
        // 生成文件名
        String suffix = "." + getSuffix(file.getOriginalFilename());
        String filename = UUID.randomUUID().toString() + suffix;
        
        // 创建目录
        File dir = new File(uploadPath);
        if (!dir.exists()) {
            dir.mkdirs();
        }
        
        // 保存文件
        File dest = new File(dir, filename);
        try {
            file.transferTo(dest);
        } catch (IOException e) {
            throw new RuntimeException("文件保存失败", e);
        }
        
        return "/uploads/" + filename;
    }
}
```

## 关联知识点

- `29_文件上传配置与原理`：上传配置
- `31_文件下载实现`：文件下载
