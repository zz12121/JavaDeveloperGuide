---
title: 泛型类面试题
tags:
  - Java/泛型
  - 原理型
  - 问答
module: 04_泛型
created: 2026-04-18
---

# 泛型类

## Q1：如何定义一个泛型类？

**A：**
在类名后用尖括号声明类型参数：
```java
public class Box<T> {
    private T value;
    public T getValue() { return value; }
}
// 使用
Box<String> box = new Box<>();
```

---

## Q2：泛型类的类型参数可以有约束吗？

**A：**
可以，使用 `extends` 关键字设置上界（Java 泛型只有上界，没有下界约束）：

```java
// 单个上界
public class NumBox<T extends Number> { ... }

// 多个上界（类必须放第一个，接口在后）
public class MultiBox<T extends Comparable<T> & Cloneable> { ... }
```

---

## Q3：为什么泛型类的静态方法不能使用类的类型参数？

**A：**
因为静态成员属于**类级别**，在 JVM 中只有一份，而泛型类型参数属于**实例级别**（每个实例的 T 可以不同）。静态方法无法确定 T 的具体类型，因此编译器禁止：

```java
public class Box<T> {
    private T value;
    
    // ❌ 编译错误：静态方法不能引用类的类型参数 T
    public static void print(T t) { ... }
    
    // ✅ 正确：静态泛型方法自己声明类型参数
    public static <U> void print(U u) { ... }
}
```

---

## Q4：泛型类可以继承普通类或其他泛型类吗？

**A：**
可以，有以下几种形式：
```java
// 继承普通类
class MyList<T> extends AbstractList<T> { ... }

// 固定父类类型参数
class StringBox extends Box<String> { ... }

// 传递自己的类型参数给父类
class WrapBox<T> extends Box<T> { ... }

// 扩展额外类型参数
class DetailBox<T, U> extends Box<T> { ... }
```

# 泛型类与泛型接口

## Q1: 泛型类和普通类在编译后有什么区别？

**A：**

### 类型擦除

Java 泛型是**编译时特性**，运行时类型信息会被擦除（Type Erasure）。

```java
// 源代码
class Box<T> {
    private T value;
    
    public T getValue() {
        return value;
    }
    
    public void setValue(T value) {
        this.value = value;
    }
}

// 编译后（等价）
class Box {
    private Object value;  // T 被替换为 Object
    
    public Object getValue() {
        return value;
    }
    
    public void setValue(Object value) {
        this.value = value;
    }
}
```

### 字节码验证

```java
// 使用 javap 验证
javap -c Box.class

Compiled from "Box.java"
class Box {
  java.lang.Object value;
    descriptor: Ljava/lang/Object;
  
  public java.lang.Object getValue();
    descriptor: ()Ljava/lang/Object;
  
  public void setValue(java.lang.Object);
    descriptor: (Ljava/lang/Object;)V
}
```

### 有界类型参数

```java
// 源代码
class NumberBox<T extends Number> {
    private T value;
    
    public T getValue() {
        return value;
    }
}

// 编译后
class NumberBox {
    private Number value;  // T 被替换为上限 Number
    
    public Number getValue() {
        return value;
    }
}
```

### 类型参数替换表

| 泛型声明 | 编译后类型 |
|---------|-----------|
| `<T>` | `Object` |
| `<T extends A>` | `A` |
| `<T extends Comparable>` | `Comparable` |
| `<T, U>` | `Object, Object` |
| `<T, U extends Number>` | `Object, Number` |

### 自动类型转换

```java
// 源代码
Box<Integer> intBox = new Box<>();
intBox.setValue(100);
int value = intBox.getValue();  // 自动拆箱

// 编译后（等价的字节码）
Box intBox = new Box();
intBox.setValue(Integer.valueOf(100));
int value = ((Integer) intBox.getValue()).intValue();  // 强制转换
```

---

## Q2: 泛型接口的两种实现方式：指定类型 vs 传递泛型参数？

**A：**

### 方式1：指定具体类型

```java
// 泛型接口
interface Container<T> {
    T get();
    void set(T value);
}

// 实现类：指定具体类型
class StringContainer implements Container<String> {
    private String value;
    
    @Override
    public String get() {
        return value;
    }
    
    @Override
    public void set(String value) {
        this.value = value;
    }
}

// 使用
Container<String> container = new StringContainer();
container.set("Hello");
String value = container.get();
```

### 方式2：传递泛型参数

```java
// 泛型接口
interface Transformer<T, R> {
    R transform(T input);
}

// 实现类：保留泛型参数
class ListToStringTransformer<T> implements Transformer<T, String> {
    @Override
    public String transform(T input) {
        return input.toString();
    }
}

// 实现类：完全保留泛型
class IdentityTransformer<T> implements Transformer<T, T> {
    @Override
    public T transform(T input) {
        return input;
    }
}

// 使用
Transformer<Integer, String> t1 = new IdentityTransformer<>();
Transformer<String, Integer> t2 = new IdentityTransformer<>();

Transformer<List<String>, Set<String>> t3 = new ListToSetTransformer<>();
```

### 两种方式对比

```java
// 方式1：非泛型实现
interface Pair<K, V> {
    K getKey();
    V getValue();
}

// 指定具体类型
class StringIntPair implements Pair<String, Integer> {
    @Override
    public String getKey() { return "key"; }
    @Override
    public Integer getValue() { return 1; }
}

// 方式2：泛型实现
class GenericPair<K, V> implements Pair<K, V> {
    private K key;
    private V value;
    
    @Override
    public K getKey() { return key; }
    @Override
    public V getValue() { return value; }
}

// 使用对比
StringIntPair p1 = new StringIntPair();  // 类型已确定
GenericPair<String, Integer> p2 = new GenericPair<>();  // 可变类型
```

### 典型应用场景

```java
// 场景1：Comparable 接口
class Person implements Comparable<Person> {
    private String name;
    
    @Override
    public int compareTo(Person other) {
        return this.name.compareTo(other.name);
    }
}

// 场景2：Factory 接口
interface Factory<T> {
    T create();
}

class StringFactory implements Factory<String> {
    @Override
    public String create() {
        return "";
    }
}

// 场景3：Repository 接口
interface Repository<T, ID> {
    T findById(ID id);
    List<T> findAll();
    void save(T entity);
    void delete(ID id);
}

class UserRepository implements Repository<User, Long> {
    @Override
    public User findById(Long id) { /* ... */ return null; }
    // ...
}
```

---

## Q3: 实现泛型接口时，如果指定了具体类型，泛型信息还存在吗？

**A：**

### 类型擦除的影响

```java
// 泛型接口
interface Handler<T> {
    void handle(T input);
}

// 实现1：指定具体类型
class StringHandler implements Handler<String> {
    @Override
    public void handle(String input) {
        System.out.println(input);
    }
}

// 实现2：保留泛型
class GenericHandler<T> implements Handler<T> {
    @Override
    public void handle(T input) {
        System.out.println(input);
    }
}
```

### 类型信息对比

```java
// 获取泛型信息
public class TypeInfo {
    
    public static void main(String[] args) {
        // StringHandler - 类型信息已丢失
        Handler<String> sHandler = new StringHandler();
        Type type = sHandler.getClass().getGenericInterfaces()[0];
        System.out.println(type);  // Handler<java.lang.String>
        
        // GenericHandler - 泛型信息存在
        Handler<Integer> gHandler = new GenericHandler<>();
        Type type2 = gHandler.getClass().getGenericInterfaces()[0];
        System.out.println(type2);  // Handler<java.lang.Integer>
    }
}
```

### 反射获取类型

```java
// 获取泛型类型参数
class TypeParameter {
    
    public static void main(String[] args) throws Exception {
        // StringHandler - 编译后等价于 Handler<Object>
        System.out.println(getGenericType(new StringHandler()));
        // 输出：class java.lang.Object（类型擦除）
        
        // GenericHandler<Integer>
        System.out.println(getGenericType(new GenericHandler<Integer>()));
        // 输出：class java.lang.Integer
    }
    
    static Class<?> getGenericType(Object obj) {
        Type[] interfaces = obj.getClass().getGenericInterfaces();
        if (interfaces.length > 0) {
            ParameterizedType pt = (ParameterizedType) interfaces[0];
            Type[] typeArgs = pt.getActualTypeArguments();
            return (Class<?>) typeArgs[0];
        }
        return Object.class;
    }
}
```

### 实际应用

```java
// Spring 中的应用
class GenericService<T> {
    protected Class<T> entityClass;
    
    @SuppressWarnings("unchecked")
    public GenericService() {
        Type type = getClass().getGenericSuperclass();
        if (type instanceof ParameterizedType) {
            ParameterizedType pt = (ParameterizedType) type;
            this.entityClass = (Class<T>) pt.getActualTypeArguments()[0];
        }
    }
}

class UserService extends GenericService<User> {
    // entityClass 会是 User.class
}

// Jackson 中的应用
class JsonResult<T> {
    public T data;
}

// 通过反射获取 T 的实际类型进行反序列化
```

---

## Q4: 为什么泛型类继承泛型类时，需要指明具体的类型参数？

**A：**

### 父类有泛型参数

```java
// 父类
class Container<T> {
    private T value;
    public T get() { return value; }
}

// ❌ 编译错误
class IntContainer extends Container {
    // 编译器假设 T = Object
}

// ✅ 正确：指定具体类型
class IntContainer extends Container<Integer> {
    // T 被绑定为 Integer
}

// ✅ 正确：保留泛型参数
class TypedContainer<T> extends Container<T> {
    // T 由实例化时指定
}
```

### 原因分析

```java
// 如果不指定类型，编译器无法确定类型参数
// 导致类型擦除后使用 Object

// 编译后的差异
class IntContainer extends Container {  // T = Object
    private Object value;
    public Object get() { return value; }
}

class IntContainer2 extends Container<Integer> {  // T = Integer
    private Integer value;
    public Integer get() { return value; }
}
```

### 多类型参数情况

```java
interface Pair<K, V> {
    K getKey();
    V getValue();
}

// ❌ 编译错误：缺少类型参数
class StringPair extends Pair {
    @Override
    public Object getKey() { return null; }
    @Override
    public Object getValue() { return null; }
}

// ✅ 正确：指定所有类型参数
class StringIntPair implements Pair<String, Integer> {
    @Override
    public String getKey() { return null; }
    @Override
    public Integer getValue() { return null; }
}

// ✅ 正确：部分指定
class NumberPair<N extends Number> implements Pair<N, String> {
    @Override
    public N getKey() { return null; }
    @Override
    public String getValue() { return null; }
}
```

### 实际使用场景

```java
// MyBatis 中的应用
class BaseMapper<T> {
    T selectById(Long id);
}

interface UserMapper extends BaseMapper<User> {
    // 自动获得 selectById(Long): User 方法
}

// 无法这样写：
// interface UserMapper extends BaseMapper  // ❌ 类型丢失

// Spring Data 中的应用
interface Repository<T, ID extends Serializable> { }

interface CrudRepository<T, ID> extends Repository<T, ID> { }

interface UserRepository extends CrudRepository<User, Long> {
    // 自动继承 User findById(Long id) 方法
}
```

---

## Q5: 泛型接口中的 default 方法可以使用泛型参数吗？

**A：**

### 可以使用

```java
interface Processor<T> {
    
    T process(T input);
    
    // ✅ default 方法可以使用泛型参数 T
    default T[] processAll(T[] inputs) {
        for (int i = 0; i < inputs.length; i++) {
            inputs[i] = process(inputs[i]);
        }
        return inputs;
    }
    
    // ✅ default 方法可以定义新的泛型方法
    default <R> List<R> convert(List<T> list, java.util.function.Function<T, R> mapper) {
        List<R> result = new java.util.ArrayList<>();
        for (T item : list) {
            result.add(mapper.apply(item));
        }
        return result;
    }
}
```

### 完整示例

```java
interface Transformer<T, R> {
    R transform(T input);
    
    // ✅ 使用接口的泛型参数
    default List<R> transformAll(List<T> inputs) {
        List<R> results = new ArrayList<>();
        for (T input : inputs) {
            results.add(transform(input));
        }
        return results;
    }
    
    // ✅ 定义新的泛型方法
    default <U> U combine(T a, T b, java.util.function.BiFunction<T, T, U> combiner) {
        return combiner.apply(a, b);
    }
}

class StringToIntTransformer implements Transformer<String, Integer> {
    @Override
    public Integer transform(String input) {
        return Integer.parseInt(input);
    }
}

// 使用
StringToIntTransformer transformer = new StringToIntTransformer();
List<String> strings = Arrays.asList("1", "2", "3");
List<Integer> ints = transformer.transformAll(strings);
System.out.println(ints);  // [1, 2, 3]
```

### 注意事项

```java
interface Dangerous<T> {
    
    T get();
    
    // ⚠️ 注意：泛型方法 <U> 与接口泛型 <T> 可以同名
    default <T> T genericMethod(T input) {
        return input;
    }
    
    // ⚠️ 注意：泛型方法遮蔽接口泛型
    default <T> void shadowDemo() {
        // 这里的 T 是泛型方法的 T，不是接口的 T
    }
}
```

---

## Q6: < T extends Number>中的 extends 既可以约束类也可以约束接口？

**A：**
### extends 的双重含义

```java
// 约束为类
class A<T extends Number> { }  // T 必须是 Number 或其子类

// 约束为接口
class B<T extends Comparable> { }  // T 必须是实现 Comparable 的类

// 约束为多个（类 + 接口）
class C<T extends Number & Comparable & Serializable> {
    // T 必须继承 Number 并实现 Comparable 和 Serializable
}
```

### 泛型边界的类型

| 声明 | 含义 |
|------|------|
| `<T>` | 无边界，T 是 Object |
| `<T extends A>` | T 是 A 或 A 的子类 |
| `<T extends A & B>` | T 继承 A 并实现 B |
| `<T extends B>` (B 是接口) | T 实现 B |

### 多边界情况

```java
// 多个边界：第一个是类，其他是接口
class MultiBound<T extends Number & Comparable & Serializable> {
    private T value;
    
    public void demo(T val) {
        // value 可以调用 Number 的方法
        double d = value.doubleValue();
        
        // value 也可以调用 Comparable 的方法
        int cmp = value.compareTo(value);
        
        // value 也可以序列化
        // ((Serializable) value).serialize();
    }
}

// ✅ 正确：Integer 满足所有条件
MultiBound<Integer> intBound = new MultiBound<>();

// ❌ 编译错误：String 不实现 Number
MultiBound<String> strBound = new MultiBound<>();
```

### 实际应用

```java
// JDK 中的应用
class Collections {
    
    public static <T extends Object & Comparable<? super T>> 
        void sort(List<T> list) {
        // T 必须实现 Comparable
        // 可以调用 compareTo 方法
    }
}

// Comparator 中的应用
@FunctionalInterface
interface Comparator<T> {
    int compare(T o1, T o2);
    
    // 多边界：实现类必须实现 Serializable 和 Comparable
    default <U extends Comparable & Serializable> 
        Comparator<U> thenComparing(Function<? super U, ? extends Comparable> keyExtractor) {
        // ...
        return null;
    }
}
```

---

## Q7: 通配符 ？、？extends T、？super T 在泛型接口实现中的区别？

**A：**

### 通配符类型

```java
// ? extends T - 上界通配符（Producer）
// ? super T - 下界通配符（Consumer）
// ? - 无界通配符
```

### ? extends T（上界）

```java
// 场景：只读取数据（生产）
List<Number> numbers = new ArrayList<>();
List<? extends Integer> ints = numbers;  // ✅

// 读取时
Integer i = ints.get(0);  // ✅ 安全
// Integer 是 Integer，所以一定是 Number

// 写入时 - 不安全
// ints.add(new Integer(1));  // ❌ 编译错误
// ints.add(new Double(1.0)); // ❌ 编译错误
// 编译器只知道容器内是 Integer 的某种子类，不确定具体类型
```

### ? super T（下界）

```java
// 场景：只写入数据（消费）
List<? super Integer> list = new ArrayList<Number>();
// list 可能是 List<Integer>、List<Number> 或 List<Object>

// 写入时 - 安全
list.add(new Integer(1));  // ✅ Integer 一定是 Integer 的父类型
list.add(1);               // ✅ 自动装箱

// 读取时 - 不安全
Object obj = list.get(0);  // ✅ 只能以 Object 读取
// Integer i = list.get(0); // ❌ 编译错误
// 编译器只知道容器内是 Integer 的某种父类型，不确定具体类型
```

### PECS 原则（Producer Extends, Consumer Super）

```java
// Producer - 生产数据给方法
public class Producer {
    
    // ✅ 使用 extends，读取数据
    public static double sum(List<? extends Number> list) {
        double sum = 0;
        for (Number n : list) {  // 只能读取
            sum += n.doubleValue();
        }
        return sum;
    }
}

// Consumer - 消费外部数据
public class Consumer {
    
    // ✅ 使用 super，写入数据
    public static void addNumbers(List<? super Integer> list) {
        list.add(1);      // ✅ 可以写入 Integer
        list.add(2);      // ✅ 自动装箱
        // Integer i = list.get(0); // ❌ 不能直接读取
    }
}

// 最佳实践
public class BestPractice {
    
    // 同时生产和消费
    public static void copy(List<? extends Number> src, 
                           List<? super Number> dst) {
        for (Number n : src) {  // 读取
            dst.add(n);          // 写入
        }
    }
}
```

### 无界通配符 ?

```java
// ? - 可以接受任何类型
public static void printList(List<?> list) {
    for (Object item : list) {
        System.out.println(item);
    }
}

// 限制：只能以 Object 读取，不能写入
public static void unsafeAdd(List<?> list) {
    // list.add("String");  // ❌ 编译错误
    // list.add(1);          // ❌ 编译错误
    // 编译器不知道容器类型，无法保证类型安全
}
```

### 泛型接口实现中的使用

```java
// 接口定义
interface Accumulator<T> {
    void accumulate(T value);
    T getResult();
}

// 使用 extends
class NumberAccumulator implements Accumulator<Double> {
    private double sum = 0;
    
    @Override
    public void accumulate(Double value) {
        sum += value;
    }
    
    @Override
    public Double getResult() {
        return sum;
    }
}

// 方法签名中使用通配符
class Processor {
    
    // 接受 Number 的某种累加器
    public static double process(Accumulator<? extends Number> accumulator) {
        // 可以获取结果（生产者）
        return accumulator.getResult().doubleValue();
    }
    
    // 接受 Integer 或其父类型的累加器
    public static void feed(Accumulator<? super Integer> accumulator) {
        // 可以写入 Integer（消费者）
        accumulator.accumulate(1);
        accumulator.accumulate(2);
    }
}
```

### 总结对比

```
┌─────────────────────────────────────────────────────────────┐
│                    通配符对比                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ? extends T                                                │
│  ├─ 读取：✅ 安全（返回 T）                                  │
│  ├─ 写入：❌ 不安全                                         │
│  └─ 用途：Producer，只读                                    │
│                                                             │
│  ? super T                                                  │
│  ├─ 读取：❌ 不安全（只能返回 Object）                       │
│  ├─ 写入：✅ 安全（可以写入 T）                             │
│  └─ 用途：Consumer，只写                                    │
│                                                             │
│  ?                                                          │
│  ├─ 读取：⚠️ 只返回 Object                                 │
│  ├─ 写入：❌ 不能写入                                       │
│  └─ 用途：只遍历，不修改                                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```
