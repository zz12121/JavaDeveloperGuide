# 🗺️ Java 基础知识地图 v1.0

> 本地图涵盖 Java 基础知识面试所需的全部知识点，共 **128个**，按模块分类。

---

## 一、Java 基础语法

| #   | 知识点                                                                    |
| --- | ---------------------------------------------------------------------- |
| 1   | Java 程序执行流程（.java → .class → JVM）                                      |
| 2   | main 方法详解（public static void main(String[] args)）                      |
| 3   | 基本数据类型（8种：byte/short/int/long/float/double/boolean/char） 基本类型默认值与占用字节数 |
| 4   | 类型转换（自动类型转换与强制类型转换）                                                    |
| 5   | 装箱与拆箱（Auto Boxing/Unboxing）                                            |
| 6   | 运算符（算术/关系/逻辑/位运算/三元运算符）                                                |
| 7   | 分支语句（if/else、switch-case、switch 新特性）                                   |
| 8   | 循环语句（for、while、do-while、foreach） break、continue、return 区别              |
| 9   | 方法（函数）定义与调用                                                            |
| 10  | 方法参数传递（值传递 vs 引用传递）                                                    |
| 11  | 可变参数（Varargs）                                                          |
| 12  | 静态导入（static import）                                                    |

---

## 二、面向对象（OOP）

| #   | 知识点                                               |
| --- | ------------------------------------------------- |
| 1   | 类与对象的关系                                           |
| 2   | 封装（Encapsulation）— 访问修饰符                          |
| 3   | 继承（Inheritance）— extends                          |
| 4   | 多态（Polymorphism）— 重写/重载                           |
| 5   | 抽象类（abstract class）vs 接口（interface）               |
| 6   | 内部类（成员内部类/静态内部类/局部内部类/匿名内部类）                      |
| 7   | 构造方法（构造函数）                                        |
| 8   | 构造代码块 vs 静态代码块 vs 普通代码块                           |
| 9   | this 关键字                                          |
| 10  | super 关键字                                         |
| 11  | static 关键字（静态属性/静态方法/静态代码块/静态内部类）                 |
| 12  | final 关键字（final 变量/方法/类）                          |
| 13  | 代码执行顺序（父类 static → 子类 static → 父类属性/构造 → 子类属性/构造） |

---

## 三、异常处理

| #   | 知识点                                                                               |
| --- | --------------------------------------------------------------------------------- |
| 1   | 异常体系（Throwable/Error/Exception）                                                   |
| 2   | Checked Exception vs Unchecked Exception                                          |
| 3   | try-catch-finally 执行顺序（含 return 场景）                                               |
| 4   | try-with-resources（自动资源关闭）                                                        |
| 5   | throw vs throws                                                                   |
| 6   | 自定义异常                                                                             |
| 7   | 常见运行时异常（NullPointerException/ClassCastException/ArrayIndexOutOfBoundsException 等） |
| 8   | 异常链（Exception Chaining）                                                           |

---

## 四、泛型

| #   | 知识点                                      |
| --- | ---------------------------------------- |
| 1   | 泛型的本质（类型参数化）                             |
| 2   | 泛型类（Generic Class）                       |
| 3   | 泛型接口（Generic Interface）                  |
| 4   | 泛型方法（Generic Method）                     |
| 5   | 泛型通配符（? extends T / ? super T / ?）       |
| 6   | 泛型擦除（Type Erasure）机制                     |
| 7   | 泛型擦除后的类型桥接                               |
| 8   | 泛型约束（不能实例化/不能创建泛型数组/静态方法不能使用类泛型）         |
| 9   | 泛型与反射结合                                  |
| 10  | PESC（Producer Extends, Consumer Super）原则 |

---

## 五、集合框架

| #   | 知识点                                                                                                                                                                                                      |
| --- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | Collection vs Collections 区别                                                                                                                                                                             |
| 2   | Iterable vs Iterator 区别                                                                                                                                                                                  |
| 3   | List 接口（ArrayList/LinkedList/Vector）                                                                                                                                                                     |
| 4   | ArrayList 底层实现（扩容机制）                                                                                                                                                                                     |
| 5   | LinkedList 底层实现（双向链表）                                                                                                                                                                                    |
| 6   | ArrayList vs LinkedList 区别与选择                                                                                                                                                                            |
| 7   | Set 接口（HashSet/LinkedHashSet/TreeSet）                                                                                                                                                                    |
| 8   | HashSet 底层实现（HashMap）                                                                                                                                                                                    |
| 9   | LinkedHashSet 实现原理（维护插入顺序）                                                                                                                                                                               |
| 10  | TreeSet 底层实现（红黑树）                                                                                                                                                                                        |
| 11  | Map 接口（HashMap/LinkedHashMap/TreeMap/Hashtable/ConcurrentHashMap）                                                                                                                                        |
| 12  | HashMap 底层实现（JDK 1.7 数组+链表 / JDK 1.8+ 红黑树） HashMap 为什么使用红黑树（链表过长时转换） HashMap 扩容机制（负载因子 0.75）HashMap put 流程（哈希计算 → 桶定位 → 链表/红黑树插入） HashMap 的 key 为什么用 String/Integer HashMap 遍历方式（keySet/entrySet/values） |
| 13  | HashMap 与 Hashtable 的区别（线程安全/同步机制）                                                                                                                                                                       |
| 14  | Queue 接口（offer/poll/peek）                                                                                                                                                                                |
| 15  | Deque 接口（双端队列）                                                                                                                                                                                           |
| 16  | fail-fast 与 fail-safe 机制                                                                                                                                                                                 |

---

## 六、IO 与 NIO

| # | 知识点 |
|---|--------|
| 1 | IO 分类（字节流/字符流、输入流/输出流） |
| 2 | 节点流 vs 处理流（装饰器模式） |
| 3 | InputStream/OutputStream 体系 |
| 4 | Reader/Writer 体系 |
| 5 | 字节流转字符流（InputStreamReader/OutputStreamWriter） |
| 6 | 缓冲流（BufferedInputStream/BufferedReader） |
| 7 | 打印流（PrintStream/PrintWriter） |
| 8 | 对象序列化（Serializable/externalizable） |
| 9 | transient 关键字 |
| 10 | serialVersionUID 的作用 |
| 11 | NIO 三大核心（Channel/Buffer/Selector） |
| 12 | Buffer 缓冲区（flip/clear/rewind） |
| 13 | Channel 通道（FileChannel/SocketChannel/ServerSocketChannel） |
| 14 | Selector 选择器（多路复用） |
| 15 | 直接内存 vs 堆内存 |
| 16 | BIO/NIO/AIO 区别（同步阻塞/同步非阻塞/异步非阻塞） |

---

## 七、注解（Annotation）

| # | 知识点 |
|---|--------|
| 1 | 注解的本质（元数据） |
| 2 | JDK 内置注解（@Override/@Deprecated/@SuppressWarnings） |
| 3 | 元注解（@Target/@Retention/@Documented/@Inherited） |
| 4 | 自定义注解 |
| 5 | 注解的提取（反射 + Annotation 接口） |
| 6 | 注解的应用场景（Spring/@Component/@Autowired/JUnit 等） |
| 7 | @FunctionalInterface 注解 |

---

## 八、反射（Reflection）

| # | 知识点 |
|---|--------|
| 1 | 反射的原理与应用场景 |
| 2 | Class 对象的获取方式（.class/forName/getClass） |
| 3 | Class 类常用方法（getDeclaredFields/getMethods/getConstructors） |
| 4 | 反射创建对象（newInstance/Constructor.newInstance） |
| 5 | 反射调用方法（Method.invoke） |
| 6 | 反射访问字段（Field.get/set） |
| 7 | 反射绕过泛型检查 |
| 8 | 反射与工厂模式/代理模式 |
| 9 | 反射的性能开销 |

---

## 九、字符串（String）

| # | 知识点 |
|---|--------|
| 1 | String 不可变性（为什么设计为不可变） |
| 2 | String、StringBuilder、StringBuffer 区别 |
| 3 | String.intern() 方法 |
| 4 | String 常用方法（length/charAt/substring/split/trim/indexOf 等） |
| 5 | String 的 + 拼接原理（编译期优化） |
| 6 | String 的哈希计算 |
| 7 | 字符串常量池（String Pool） |
| 8 | new String("xxx") 创建几个对象 |

---

## 十、Lambda 与函数式编程

| # | 知识点 |
|---|--------|
| 1 | Lambda 表达式语法 |
| 2 | 函数式接口（@FunctionalInterface） |
| 3 | 四大核心函数式接口（Consumer/Supplier/Predicate/Function） |
| 4 | 方法引用（Method Reference） |
| 5 | 构造器引用 |
| 6 | Lambda 表达式作用域（访问局部变量/final 限制） |

---

## 十一、Stream API

| # | 知识点 |
|---|--------|
| 1 | Stream 的创建（Collection.stream/Arrays.stream/Stream.of） |
| 2 | 中间操作（filter/map/flatMap/distinct/sorted/limit/skip） |
| 3 | 终端操作（collect/forEach/count/min/max/sum/reduce） |
| 4 | Optional 与 Stream 结合 |
| 5 | 并行流（parallelStream）原理 |
| 6 | Stream 惰性求值 |
| 7 | Stream 的性能考虑 |

---

## 十二、其他核心概念

| # | 知识点 |
|---|--------|
| 1 | == vs equals() |
| 2 | hashCode() 与 equals() 关系 |
| 3 | Object 类方法与 Objects 工具类 |
| 4 | 深拷贝 vs 浅拷贝 |
| 5 | clone() 方法与 Cloneable 接口 |
| 6 | Comparable vs Comparator |
| 7 | 枚举类（enum） |
| 8 | 枚举实现接口 |
| 9 | 枚举单例模式 |
| 10 | System 类常用方法（arraycopy/exit/gc/currentTimeMillis） |
| 11 | Math 类常用方法 |
| 12 | Random 类的线程安全问题 |
| 13 | ThreadLocalRandom（高性能随机数） |
| 14 | BigDecimal 浮点数精度问题 |
| 15 | Date/LocalDateTime/Instant 时间处理 |
| 16 | Arrays 工具类常用方法 |

---

## 十三、Java 新特性（JDK 8~21）

| # | 知识点 |
|---|--------|
| 1 | 接口默认方法与静态方法（JDK 8） |
| 2 | 接口私有方法（JDK 9） |
| 3 | var 局部变量类型推断（JDK 10） |
| 4 | 集合工厂方法（List.of/Set.of/Map.of）（JDK 9） |
| 5 | Optional 类（空值处理）（JDK 8） |
| 6 | String 改进（join/lines/strip/repeat/isBlank）（JDK 11） |
| 7 | Switch 表达式（JDK 12 预览/14 正式） |
| 8 | Text Blocks 文本块（JDK 13 预览/15 正式） |
| 9 | Pattern Matching for instanceof（JDK 14 预览/16 正式） |
| 10 | Records 记录类（JDK 14 预览/16 正式） |
| 11 | Sealed Classes 密封类（JDK 15 预览/17 正式） |
| 12 | 模式匹配 for switch（JDK 21 正式） |

---

*最后更新：2026-4-21*
