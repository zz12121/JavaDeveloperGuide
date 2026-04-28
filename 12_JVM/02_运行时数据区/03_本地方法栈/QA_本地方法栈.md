
# 本地方法栈

## Q1：什么是本地方法栈？

**A**：本地方法栈（Native Method Stack）是 JVM 管理的内存区域，专门用于执行 native 方法（通过 JNI 调用 C/C++ 代码）。它与 Java 虚拟机栈类似，但服务于不同的方法类型。

---

## Q2：本地方法栈和 Java 虚拟机栈有什么区别？

**A**：

| 区别 | Java 虚拟机栈 | 本地方法栈 |
|------|---------------|------------|
| **执行方法** | Java 字节码方法 | native 方法 |
| **实现方式** | JVM 规范强制实现 | JVM 规范允许不实现 |
| **HotSpot** | 必须实现 | 与 Java 栈合并实现 |

---

## Q3：哪些方法会使用本地方法栈？

**A**：Java 中的 native 方法会使用本地方法栈：
- `Object.hashCode()`、`Object.clone()` 等基础方法
- `Thread.start()` 等线程相关方法
- `System.loadLibrary()` 加载本地库
- `System.arraycopy()` 数组操作
- `Thread.currentTimeMillis()`

---

## Q4：本地方法栈会抛出哪些异常？

**A**：与 Java 虚拟机栈一样：
- **StackOverflowError**：线程请求的栈深度超过最大限制
- **OutOfMemoryError**：无法申请到足够内存扩展栈

---

## Q5：HotSpot 对本地方法栈的实现？

**A**：HotSpot VM 将本地方法栈和 Java 虚拟机栈**合并实现**，使用同一个参数控制：
- `-Xss` 同时控制两者的大小
- 但概念上它们是逻辑分离的两个区域

---

## Q6：为什么需要 native 方法？

**A**：native 方法的用途：
1. **调用系统级 API**：访问操作系统底层功能
2. **性能优化**：某些操作用 C/C++ 实现更高效
3. **复用已有代码**：集成现有的 C/C++ 库
4. **硬件交互**：直接与硬件通信

---

## Q7：native 方法的注意事项？

**A**：
- native 方法不遵守 Java 的异常处理机制
- native 方法的堆栈跟踪可能不完整
- 本地库需要正确加载（`System.loadLibrary`）
- 不同平台的 native 库可能不同
