
# JDK / JRE / JVM 区别

## Q1：JDK、JRE、JVM 三者的区别是什么？

**A**：
- **JDK**（Java Development Kit）：Java 开发工具包，**包含 JRE + 开发工具**（javac、jdb、jar、javadoc 等）
- **JRE**（Java Runtime Environment）：Java 运行时环境，**包含 JVM + 核心 Java 类库**
- **JVM**（Java Virtual Machine）：Java 虚拟机，负责**执行字节码**

**关系**：**JDK 包含 JRE 包含 JVM**

---

## Q2：为什么说 Java 是"一次编写，到处运行"？

**A**：因为 JVM 层面的抽象。`.java` 源码被 `javac` 编译成平台无关的 **.class 字节码**，不同平台的 JVM 将字节码翻译为对应平台的机器码执行。只要目标平台有对应的 JVM，同一个 .class 文件就能运行。

---

## Q3：JDK 8 之后有哪些重要 LTS 版本？

**A**：

| 版本 | 重要变化 |
|------|---------|
| JDK 8 | Lambda、Stream、PermGen -> Metaspace |
| JDK 11 | LTS，ZGC 实验，HTTP Client API，JFR 免费 |
| JDK 17 | LTS，密封类，ZGC 正式发布 |
| JDK 21 | LTS，虚拟线程（Project Loom），分代 ZGC |

---

## Q4：生产服务器应该装 JDK 还是 JRE？

**A**：通常只需要 JRE。但如果需要热调试（动态 attach、热部署等），则需要 JDK。Docker 容器化部署通常使用 **Eclipse Temurin / Amazon Corretto** 等 JDK 精简镜像。


