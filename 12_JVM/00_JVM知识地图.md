## 一、JVM体系结构

| #   | 知识点                                             |
| --- | ----------------------------------------------- |
| 1   | **JVM整体架构**（ClassLoader + 运行时数据区 + 执行引擎 + 本地接口） |
| 2   | **JDK、JRE和JVM 区别**                              |
| 3   | **JVM运行流程**（.java → .class → 类加载 → 执行）          |

---

## 二、运行时数据区（Runtime Data Area）

| #   | 知识点                                                     |
| --- | ------------------------------------------------------- |
| 1   | **程序计数器（PC Register）** — 记录字节码位置，native方法为空，唯一不抛OOM的区域  |
| 2   | **Java虚拟机栈（JVM Stack）** — 栈帧：局部变量表 + 操作数栈 + 动态链接 + 返回地址 |
| 3   | **本地方法栈（Native Stack）** — native方法服务，HotSpot将二者合一       |
| 4   | **堆（Heap）** — 对象实例 + 数组 + 字符串常量池（JDK7+），OOM主要区域         |
| 5   | **方法区（Method Area）** — 类信息 + 常量 + 静态变量 + JIT代码缓存        |
| 6   | **运行时常量池** — Class文件常量池的运行时表示，JDK6在方法区，JDK7+在堆          |
| 7   | **直接内存（Direct Memory）** — NIO堆外内存，不属于运行时数据区但常连问         |

---

## 三、对象

| #   | 知识点                                                       |
| --- | --------------------------------------------------------- |
| 1   | **new对象完整流程** — 类加载检查 → 分配内存 → 初始化零值 → 设置对象头 → 执行init     |
| 2   | **内存分配方式** — 指针碰撞（Bump the Pointer）+ 空闲列表（Free List）      |
| 3   | **内存分配并发问题** — CAS + TLAB解决                               |
| 4   | **对象的内存布局** — 对象头（Mark Word + Class Pointer）+ 实例数据 + 对齐填充 |
| 5   | **句柄访问** — 堆中句柄池，间接访问                                     |
| 6   | **直接指针访问（HotSpot采用）** — 对象引用直接指向堆中对象                      |

---

## 四、字符串与常量池

| #   | 知识点                                                  |
| --- | ---------------------------------------------------- |
| 1   | **字符串常量池位置演变** — JDK6 PermGen → JDK7堆 → JDK8堆（元空间移除） |
| 2   | **String.intern()方法** — JDK6/7/8行为差异，大量intern的坑      |
| 3   | **new String()创建对象数** — JDK6创建2个，JDK7+可能1个           |
| 4   | **字符串拼接优化** — 编译期常量合并，StringBuilder优化                |
| 5   | **StringTable** — Hashtable实现，JDK7+可扩容               |
| 6   | **Compact Strings** — Latin1/UTF16编码优化               |
| 7   | **Class文件常量池** — 编译期确定的字面量和符号引用                      |
| 8   | **运行时常量池** — Class文件常量池的运行时部分，支持动态扩展                 |

---

## 五、类加载机制

|#|知识点|
|---|---|
|1|**类加载5阶段** — 加载 → 验证 → 准备 → 解析 → 初始化|
|2|**加载阶段** — 获取字节流 → 静态结构 → Class对象|
|3|**验证阶段** — 文件格式 + 元数据 + 字节码 + 符号引用|
|4|**准备阶段** — static分配默认值（final直接赋初值）|
|5|**解析阶段** — 符号引用 → 直接引用，类/接口/字段/方法解析|
|6|**初始化阶段** — 执行<clinit>，赋真正初值，线程安全|
|7|**三层类加载器** — Bootstrap / Extension / Application|
|8|**双亲委派模型** — 优先父加载器，保证唯一性和安全性|
|9|**打破双亲委派** — Tomcat / JDBC / SPI / 热部署 / 模块化|
|10|**loadClass源码** — 双亲委派实现：findLoadedClass → 父加载 → findClass|
|11|**Class.forName vs ClassLoader** — 初始化区别|
|12|**自定义类加载器** — 继承ClassLoader，重写findClass|
|13|**线程上下文类加载器** — SPI打破双亲委派的机制|
|14|**类卸载条件** — 无实例 + 无Class引用 + ClassLoader被卸载|
|15|**OSGi类加载器架构** — 模块化类加载，面试加分项|
|16|**JPMS模块化系统** — JDK9+，module-info.java，exports/requires|

---

## 六、字节码与执行引擎

|#|知识点|
|---|---|
|1|**Class文件结构** — 魔数 + 版本 + 常量池 + 访问标志 + 类索引 + ...|
|2|**字节码指令集** — 加载存储/操作数栈/算术/类型转换/对象创建/控制转移/...|
|3|**javap工具使用** — -c -v -p 查看字节码|
|4|**常见字节码指令** — aload_/astore_/getfield/putfield/invokevirtual/...|
|5|**ASM** — 树访问器 + Visitor模式，编译时字节码操作|
|6|**Javassist** — 源代码级操作字节码|
|7|**ByteBuddy** — 运行时字节码生成，Agent/Mock框架|
|8|**cglib** — 基于ASM，Spring/Hibernate使用|
|9|**解释执行 vs 编译执行** — 字节码 → 机器码，两种执行方式|
|10|**JIT编译器** — C1（客户端编译器）+ C2（服务端编译器）|
|11|**分层编译** — C0解释 → C1编译 → C2编译，4个Level|
|12|**热点代码探测** — 方法调用计数器 + 回边计数器|
|13|**OSR栈上替换** — 循环编译时，正在执行的方法切换到编译版本|
|14|**JIT编译日志** — -XX:+PrintCompilation|
|15|**Code Cache作用** — 存储JIT编译后的机器码|
|16|**Code Cache配置** — -XX:InitialCodeCacheSize / -XX:ReservedCodeCacheSize|
|17|**Code Cache满** — JIT编译关闭，性能下降|
|18|**Code Cache回收** — JDK9+，非冷门代码可回收|

---

## 七、垃圾回收（GC）

|#|知识点|
|---|---|
|1|**判断对象是否可回收** — 引用计数法（缺陷）+ 可达性分析（主流）|
|2|**GC Roots有哪些** — 虚拟机栈/本地栈 + 方法区静态/常量 + synchronized + 本地方法/内部引用|
|3|**四种引用类型** — 强/软/弱/虚，回收时机不同|
|4|**引用计数法缺陷** — 循环引用无法回收|
|5|**标记-清除算法** — 标记+清除，缺点产生碎片|
|6|**复制算法** — 分两块/分代，缺点浪费空间，用于年轻代|
|7|**标记-整理算法** — 标记+移动，缺点移动成本，用于老年代|
|8|**分代收集理论** — 弱分代假说 + 强分代假说 + 跨代引用|
|9|**GC日志分析** — JDK8格式 vs JDK9+格式|
|10|**Serial收集器** — 单线程，Client模式，年轻代，复制|
|11|**ParNew收集器** — Serial多线程版本，Server年轻代，配合CMS|
|12|**Parallel Scavenge** — 吞吐量优先，JDK8年轻代默认，自适应调节|
|13|**Serial Old收集器** — 单线程，老年代，标记-整理|
|14|**Parallel Old收集器** — 多线程，JDK8老年代默认|
|15|**CMS收集器** — 4阶段，低停顿，并发标记清除，浮动垃圾+碎片|
|16|**G1收集器** — Region分代，可预测停顿，JDK9+默认|
|17|**ZGC** — 染色指针，并发，极低停顿（<1ms），TB级|
|18|**Shenandoah** — OpenJDK，低停顿，不依赖JDK版本|
|19|**Epsilon** — 无操作GC，性能测试|
|20|**收集器组合** — ParNew+CMS / Parallel+ParallelOld / G1 / ZGC|
|21|**如何选择收集器** — 吞吐量优先/低延迟优先/大内存|
|22|**Minor GC** — 清理年轻代，Eden满触发，频率高停顿短|
|23|**Major GC** — 清理老年代，CMS特有概念，常伴随Minor GC|
|24|**Full GC** — 清理整个堆+方法区，最耗时，应尽量避免|
|25|**对象晋升老年代条件** — 年龄阈值 + 动态年龄判断 + 大对象 + Survivor空间不足|
|26|**对象年龄更新** — Minor GC复制时age++|
|27|**空间分配担保** — JDK7+ HandlePromotionFailure，Minor GC前检查|

## 八、性能调优

|#|知识点|
|---|---|
|1|**堆参数** — -Xms / -Xmx / -Xmn / -XX:NewRatio / -XX:SurvivorRatio|
|2|**方法区参数** — -XX:MetaspaceSize / -XX:MaxMetaspaceSize|
|3|**GC参数** — -XX:+UsexxxGC / -XX:+PrintGCDetails / -XX:+PrintGCDateStamps|
|4|**TLAB参数** — -XX:+UseTLAB / -XX:TLABSize / -XX:+ResizeTLAB|
|5|**直接内存参数** — -XX:MaxDirectMemorySize|
|6|**OOM时导出堆** — -XX:+HeapDumpOnOutOfMemoryError / -XX:HeapDumpPath|
|7|**吞吐量优先调优** — Parallel Scavenge + Parallel Old，自适应策略|
|8|**延迟优先调优** — G1/ZGC，-XX:MaxGCPauseMillis|
|9|**内存占用优先** — 尽量小，使用压缩指针等|
|10|**GC调优步骤** — 确定目标 → 监控GC → 分析日志 → 调整参数 → 验证|
|11|**G1调优参数** — -XX:MaxGCPauseMillis / -XX:G1HeapRegionSize / -XX:InitiatingHeapOccupancyPercent|

---

## 九、故障排查

|#|知识点|
|---|---|
|1|**jps** — 查看Java进程|
|2|**jstat** — 查看GC统计，-gcutil / -gc / -gccapacity|
|3|**jinfo** — 查看/修改JVM参数|
|4|**jmap** — 导出堆dump，查看内存使用，histogram|
|5|**jstack** — 线程dump，死锁检测|
|6|**jcmd** — 综合工具，VM.flags / GC.class_histogram等|
|7|**MAT** — 堆分析，Dominator Tree，泄漏检测|
|8|**Arthas** — dashboard / trace / ognl / jad / redefine|
|9|**async-profiler** — CPU/内存采样，低开销|
|10|**Btrace** — 动态跟踪，生产环境热修复|
|11|**GCEasy** — GC日志在线分析|
|12|**GCViewer** — 本地GC日志分析|
|13|**OOM排查** — heapdump + MAT分析，泄漏 vs 溢出|
|14|**CPU 100%排查** — top → jstack → 定位代码|
|15|**死锁排查** — jstack检测死锁，wait/notify/synchronized|
|16|**内存泄漏排查** — MAT Dominator Tree，WeakHashMap，缓存未清理|
|17|**元空间OOM** — 类加载过多，动态代理/JSP/反射|
|18|**晋升失败** — Promotion Failed，Survivor空间不足|
|19|**抖动问题** — 频繁GC，对象朝生夕死，存活判断不准|
|20|**堆溢出排查** — HeapDump+MAT分析，泄漏vs溢出，常见泄漏场景|
|21|**栈溢出** — StackOverflowError，递归过深/方法调用层级太深，-Xss参数调优|
|22|**直接内存溢出** — OutOfMemoryError: Direct buffer memory，NIO未释放，MaxDirectMemorySize|
|23|**OOM排查流程** — 确认OOM类型→导出dump→MAT分析→定位根因→修复验证|
|24|**OOM预防措施** — 合理设置堆大小、监控告警、代码审查缓存/连接池、压力测试|

---

## 十、逃逸分析进阶

|#|知识点|
|---|---|
|1|**逃逸分析基础** — 方法逃逸 / 线程逃逸 / 不逃逸|
|2|**栈上分配** — 标量替换，对象拆散成标量放栈|
|3|**同步消除** — 锁消除，无竞争时消除synchronized|
|4|**标量替换** — 对象替换为成员变量|
|5|**HotSpot实现** — 逃逸分析已实现，但栈上分配不完整（通过标量替换实现）|

---

## 十一、其他重要知识点

|#|知识点|
|---|---|
|1|**NMT内存追踪** — -XX:NativeMemoryTracking=summary，jcmd VM.native_memory|
|2|**JFR** — JDK Flight Recorder，低开销持续诊断|
|3|**GraalVM** — 高性能语言虚拟机，AOT编译|
|4|**Docker/K8s容器JVM参数** — 容器内JVM配置注意事项|
|5|**压缩指针** — -XX:+UseCompressedOops / -XX:+UseCompressedClassPointers|
|6|**Object Alignment** — -XX:ObjectAlignmentInBytes，默认8字节对齐|

---

## 十二、JDK版本演进

|#|知识点|
|---|---|
|1|**JDK8 vs JDK11** — PermGen到Metaspace, Nashorn移除, ZGC支持, 容器感知|
|2|**JDK11新特性** — HTTP Client API, var推断, String增强, ZGC实验, JFR免费|
|3|**JDK17新特性** — 密封类, ZGC正式版, 模式匹配switch, Text Blocks|
|4|**JDK21新特性** — 虚拟线程(Project Loom), 分代ZGC, Record Patterns|
|5|**默认垃圾收集器演进** — Serial到Parallel到G1到ZGC，各版本默认GC变化|
|6|**其他版本差异** — GC日志格式(JDK8到JDK9+), JVM参数演进, JFR历史|
