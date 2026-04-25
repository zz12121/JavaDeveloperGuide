---
title: NIO Path与Files API
tags:
  - Java/IO
  - 工具类
  - NIO
module: 06_IO与NIO
created: 2026-04-25
---

# NIO Path与Files API

## Path 接口（JDK 7+）

### 什么是 Path

- Path 是 `java.nio.file.Path` 接口，用于替代 `java.io.File`
- File 和 Path 的转换：`path.toFile()` / `file.toPath()`

### Path 的创建

```java
Path p1 = Paths.get("data.txt");                        // 相对路径
Path p2 = Paths.get("C:", "Users", "admin", "file.txt"); // 多参数
Path p3 = Paths.get("file:///data.txt");                  // URI
Path p4 = Path.of("data.txt");                            // JDK 11+ 等价写法
```

### Path 的常用方法

| 方法 | 说明 |
|------|------|
| `path.getFileName()` | 获取文件名（含扩展名） |
| `path.getParent()` | 获取父目录 |
| `path.getRoot()` | 获取根路径（Windows: `C:\`） |
| `path.isAbsolute()` | 是否绝对路径 |
| `path.toAbsolutePath()` | 转为绝对路径 |
| `path.resolve(other)` | 拼接子路径（推荐） |
| `path.resolveSibling(other)` | 拼接兄弟路径 |
| `path.subpath(0, 2)` | 获取子路径 |
| `path.normalize()` | 规范化路径（去除 `.` 和 `..`） |
| `path.startsWith(other)` | 是否以某路径开头 |
| `path.endsWith(other)` | 是否以某路径结尾 |

### Path 与 File 的对比

| 功能 | File | Path |
|------|------|------|
| 获取路径 | `file.getPath()` | `path` |
| 获取绝对路径 | `file.getAbsolutePath()` | `path.toAbsolutePath()` |
| 获取规范路径 | `file.getCanonicalPath()` | `path.toRealPath()` |
| 父路径 | `file.getParent()` | `path.getParent()` |
| 文件名 | `file.getName()` | `path.getFileName()` |
| 是否绝对路径 | `file.isAbsolute()` | `path.isAbsolute()` |

> **推荐使用 Path**，Path 是不可变对象，线程安全，且 API 更丰富。

## Files 工具类（JDK 7+）

### 文件读写

```java
// 读
byte[] data = Files.readAllBytes(Path.of("config.json"));           // 小文件读全部
String content = Files.readString(Path.of("config.json"));            // JDK 11+
List<String> lines = Files.readAllLines(Path.of("log.txt"));         // 逐行读取

// 流式大文件处理（推荐）
try (Stream<String> lines = Files.lines(Path.of("big.log"))) {
    lines.filter(l -> l.contains("ERROR"))
         .forEach(System.out::println);
}

// 写
Files.writeString(Path.of("out.txt"), "Hello", StandardOpenOption.CREATE);
Files.write(Path.of("out.txt"), lines, StandardOpenOption.CREATE);
```

### 文件和目录操作

```java
// 创建
Files.createFile(Path.of("new.txt"));
Files.createDirectories(Path.of("a/b/c/d"));  // 级联创建

// 复制
Files.copy(src, dest);                                    // 覆盖
Files.copy(src, dest, StandardCopyOption.REPLACE_EXISTING,
                        StandardCopyOption.COPY_ATTRIBUTES);

// 移动
Files.move(src, dest, StandardCopyOption.REPLACE_EXISTING);

// 删除
Files.delete(path);              // 不存在则抛异常
Files.deleteIfExists(path);      // 不存在则静默跳过

// 判断
Files.exists(path);
Files.notExists(path);           // 不存在（注意：exists和notExists都不确定时返回false）
Files.isSameFile(path1, path2);
Files.isHidden(path);
Files.isDirectory(path);
Files.isRegularFile(path);
```

### 文件属性

```java
// 基本属性
long size = Files.size(path);
FileTime time = Files.getLastModifiedTime(path);

// 设置属性
Files.setLastModifiedTime(path, FileTime.fromMillis(time));

// 权限（JDK 12+）
Files.setPosixFilePermissions(path, PosixFilePermissions.fromString("rwxr-xr-x"));
```

### 目录遍历

```java
// 遍历目录（不递归）
Files.list(Path.of("dir"))
     .filter(Files::isRegularFile)
     .forEach(System.out::println);

// 深度遍历
Files.walk(Path.of("project"), 3)       // 最大深度3层
     .filter(Files::isRegularFile)
     .forEach(System.out::println);

// 查找（带条件）
Path result = Files.find(Path.of("."),
    5,                                      // 最大深度
    (p, attr) -> attr.size() > 1024 * 1024  // 大于1MB
).findFirst().orElse(null);

// FileVisitor（完整控制）
Files.walkFileTree(Path.of("dir"), new SimpleFileVisitor<>() {
    @Override
    public FileVisitResult visitFile(Path file, BasicFileAttributes attrs) {
        System.out.println(file);
        return CONTINUE;
    }

    @Override
    public FileVisitResult postVisitDirectory(Path dir, IOException exc) {
        System.out.println("退出: " + dir);
        return CONTINUE;
    }
});
```

## FileVisitor 接口（JDK 7+）

### 四个回调方法

| 方法 | 何时调用 | 返回值 |
|------|---------|--------|
| `preVisitDirectory()` | 进入目录前 | CONTINUE / SKIP_SUBTREE / SKIP_SIBLINGS / TERMINATE |
| `visitFile()` | 访问文件时 | CONTINUE / TERMINATE |
| `postVisitDirectory()` | 离开目录后 | CONTINUE / TERMINATE |
| `visitFileFailed()` | 访问失败时 | CONTINUE / TERMINATE / SKIP_SIBLINGS |

### SimpleFileVisitor（常用实现）

```java
// 递归删除（类似 rm -rf）
Files.walkFileTree(Path.of("target"), new SimpleFileVisitor<>() {
    @Override
    public FileVisitResult visitFile(Path file, BasicFileAttributes attrs) {
        Files.delete(file);
        return CONTINUE;
    }

    @Override
    public FileVisitResult postVisitDirectory(Path dir, IOException exc) {
        Files.delete(dir);
        return CONTINUE;
    }
});

// 统计文件数量和大小
Files.walkFileTree(Path.of("project"), new SimpleFileVisitor<>() {
    private long count = 0;
    private long size = 0;

    @Override
    public FileVisitResult visitFile(Path file, BasicFileAttributes attrs) {
        count++;
        size += attrs.size();
        return CONTINUE;
    }
});
```

## WatchService 监控文件变化（JDK 7+）

```java
WatchService watcher = FileSystems.getDefault().newWatchService();
Path path = Paths.get("src");

// 注册监听事件
path.register(watcher,
    StandardWatchEventKinds.ENTRY_CREATE,   // 新建
    StandardWatchEventKinds.ENTRY_MODIFY,  // 修改
    StandardWatchEventKinds.ENTRY_DELETE);  // 删除

while (true) {
    WatchKey key = watcher.take();  // 阻塞等待

    for (WatchEvent<?> event : key.pollEvents()) {
        WatchEvent.Kind<?> kind = event.kind();
        Path changed = (Path) event.context();
        System.out.println(kind + ": " + changed);
    }

    key.reset();  // 重置 key，继续监听
}
```

## 关联知识点

