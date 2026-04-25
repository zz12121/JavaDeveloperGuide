---
title: NIO Path与Files API面试题
tags:
  - Java/IO
  - 工具类
  - 问答
module: 06_IO与NIO
created: 2026-04-25
---

# NIO Path与Files API

## Q1：Path 和 File 有什么区别？为什么要用 Path 替代 File？

**A：**

| 维度 | java.io.File | java.nio.file.Path |
|------|:---:|:---:|
| 类型 | 具体类 | 接口（可扩展） |
| 可变性 | 可变 | 不可变（线程安全） |
| 符号链接 | 处理复杂 | 有专门方法处理 |
| URI 支持 | 一般 | 原生 URI 支持 |
| API 丰富度 | 有限 | 非常丰富 |
| 路径拼接 | 字符串拼接 | `resolve()` 方法 |
| JDK 版本 | 1.0 | 1.7+ |

```java
// File 的问题：路径拼接容易出错
File f1 = new File("dir", "sub");
File f2 = new File(f1, "file.txt");

// Path 的优势：API 丰富且直观
Path p1 = Paths.get("dir", "sub");
Path p2 = p1.resolve("file.txt");           // resolve
Path p3 = p1.resolveSibling("other.txt");   // resolveSibling
Path p4 = p1.resolve("..").normalize();    // 规范化

// Path 和 File 互转
Path path = file.toPath();
File file = path.toFile();
```

> **结论**：Java 7+ 推荐使用 `Path` 和 `Files`，File 主要是为了向后兼容。

---

## Q2：Files.readAllLines() 和 Files.lines() 有什么区别？

**A：**

| 方法 | 特点 | 适用场景 |
|------|------|---------|
| `readAllLines()` | 一次性读入内存（返回 List） | 小文件（<100MB） |
| `lines()` | 流式处理（返回 Stream<String>） | **大文件、需过滤/处理** |

```java
// readAllLines：全部加载到内存
List<String> lines = Files.readAllLines(Path.of("config.txt"));
// 内存占用 = 文件大小 × 2（每行String对象开销大）

// lines：流式处理，自动关闭资源
try (Stream<String> lines = Files.lines(Path.of("big.log"))) {
    long errorCount = lines
        .filter(l -> l.contains("ERROR"))
        .count();  // 逐行处理，内存占用极低
}
```

**重要区别**：`Files.lines()` 返回的 Stream 必须使用 try-with-resources 关闭，否则文件句柄会泄漏。

---

## Q3：Files.walk() 和 Files.find() 有什么区别？

**A：**

| 方法 | 功能 | 过滤条件 |
|------|------|---------|
| `Files.walk()` | 深度遍历，返回 Path | 无内置过滤，需 `.filter()` |
| `Files.find()` | 深度遍历，返回 Path | **支持 BasicFileAttributes 过滤** |

```java
// walk：遍历所有文件，filter在外层
Files.walk(Path.of("project"))
     .filter(p -> p.toString().endsWith(".java"))
     .forEach(System.out::println);

// find：过滤在遍历时进行（更高效，减少遍历次数）
Files.find(Path.of("project"),
    10,                                             // 最大深度
    (path, attrs) -> path.toString().endsWith(".java"),  // 用属性过滤
    FileVisitOption.FOLLOW_LINKS                     // 跟随符号链接
).forEach(System.out::println);
```

---

## Q4：Files.walkFileTree() 的 FileVisitor 四种返回值分别代表什么？

**A：**

| 返回值 | 含义 |
|--------|------|
| `FileVisitResult.CONTINUE` | 继续遍历（正常行为） |
| `FileVisitResult.SKIP_SIBLINGS` | 跳过同级的其他文件/目录 |
| `FileVisitResult.SKIP_SUBTREE` | 不进入当前目录（仅限 preVisitDirectory） |
| `FileVisitResult.TERMINATE` | 立即终止整个遍历 |

```java
// 典型应用：只处理特定目录下的文件
Files.walkFileTree(Path.of("src"), new SimpleFileVisitor<>() {
    @Override
    public FileVisitResult preVisitDirectory(Path dir, BasicFileAttributes attrs) {
        if (dir.getFileName().toString().equals("test")) {
            return SKIP_SUBTREE;  // 跳过 test 目录
        }
        return CONTINUE;
    }
});
```

---

## Q5：WatchService 能监控文件内容变化吗？有什么限制？

**A：**

**WatchService 只能监控文件系统事件，不能监控文件内容变化。**

可监控的事件类型：
- `ENTRY_CREATE` — 新建文件/目录
- `ENTRY_MODIFY` — 文件/目录被修改
- `ENTRY_DELETE` — 删除文件/目录
- `OVERFLOW` — 事件过多溢出（需特殊处理）

**限制**：
1. **跨平台差异大**：Linux 使用 inotify（高效），Windows 使用 ReadDirectoryChangesW，macOS 使用 FSEvents
2. **不能监控网络文件系统**（NFS、SMB 等）
3. **不能监控远程文件**
4. **不支持递归监听子目录**：需要手动为每个子目录注册 WatchKey
5. **需要手动处理 OVERFLOW**：事件堆积时要重新扫描目录

```java
// 递归注册 WatchService 的方式
private void registerAll(Path dir, WatchService ws) throws IOException {
    Files.list(dir).forEach(path -> {
        if (path.toFile().isDirectory()) {
            registerAll(path, ws);  // 递归注册
        }
        path.register(ws, ENTRY_CREATE, ENTRY_DELETE, ENTRY_MODIFY);
    });
}
```

---

## Q6：Files.delete() 和 Files.deleteIfExists() 有什么区别？

**A：**

| 方法 | 行为 | 适用场景 |
|------|------|---------|
| `delete()` | 文件不存在则抛 `NoSuchFileException` | 确认文件一定存在时才用 |
| `deleteIfExists()` | 文件不存在则静默跳过 | **推荐使用**，安全幂等 |

```java
// delete()：不存在就抛异常
try {
    Files.delete(Path.of("file.txt"));
} catch (NoSuchFileException e) {
    // 文件不存在，抛异常
}

// deleteIfExists()：不存在就跳过（幂等操作）
boolean deleted = Files.deleteIfExists(Path.of("file.txt"));
// deleted = true 表示删除成功，false 表示原本就不存在
```

---

## Q7：Files.exists() 的坑是什么？

**A：**

`Files.exists()` 的一个经典陷阱：**同一时间检查和不存在的返回结果是假阳性**。

```java
// 经典的 TOCTOU（Time-Of-Check to Time-Of-Use）问题：
if (Files.exists(path)) {        // 时刻T1：检查存在
    Files.delete(path);           // 时刻T2：删除
}
// 如果T1~T2之间文件被其他线程删除了，就会抛异常
```

**正确做法**：直接操作，不需要先检查。

```java
// 错误：TOCTOU 竞态条件
if (Files.exists(path)) {
    Files.delete(path);
}

// 正确：直接删除（deleteIfExists 本身就是幂等的）
Files.deleteIfExists(path);

// 正确：用 Atomic 文件操作（JDK 19+）
Files.move(from, to, StandardCopyOption.REPLACE_EXISTING,
                     StandardCopyOption.ATOMIC_MOVE);
```
