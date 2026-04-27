
# ThreadLocal使用

## 核心结论

`ThreadLocal<T>` 提供线程级别的变量副本，核心 API 只有三个：`set()`、`get()`、`remove()`。每个线程对同一个 `ThreadLocal` 的读写互不影响，实现**以空间换时间**的线程安全策略。

## 深度解析

### 基本 API

```java
// 1. 创建 ThreadLocal
private static final ThreadLocal<String> USER_CONTEXT = new ThreadLocal<>();

// 2. 带初始值的 ThreadLocal（推荐）
private static final ThreadLocal<Integer> COUNTER = ThreadLocal.withInitial(() -> 0);

// 3. 设置值
USER_CONTEXT.set("admin");

// 4. 获取值（首次 get 返回 initialValue() 的值）
String user = USER_CONTEXT.get();  // "admin"

// 5. 移除值（重要！尤其在线程池中）
USER_CONTEXT.remove();
```

### 两种初始化方式

```java
// 方式一：重写 initialValue()（JDK 8 之前）
private static final ThreadLocal<SimpleDateFormat> DATE_FORMAT = 
    new ThreadLocal<SimpleDateFormat>() {
        @Override
        protected SimpleDateFormat initialValue() {
            return new SimpleDateFormat("yyyy-MM-dd");
        }
    };

// 方式二：withInitial()（JDK 8+，推荐）
private static final ThreadLocal<SimpleDateFormat> DATE_FORMAT = 
    ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));
```

### 典型使用模式

```java
// 模式一：请求上下文传递（Web 应用）
public class UserContext {
    private static final ThreadLocal<User> CURRENT_USER = new ThreadLocal<>();

    public static void set(User user) { CURRENT_USER.set(user); }
    public static User get() { return CURRENT_USER.get(); }
    public static void clear() { CURRENT_USER.remove(); }
}

// Filter 中设置
public void doFilter(request, response, chain) {
    try {
        UserContext.set(parseUser(request));
        chain.doFilter(request, response);
    } finally {
        UserContext.clear();  // 必须清理！
    }
}
```

```java
// 模式二：线程安全的工具类（SimpleDateFormat 线程不安全）
private static final ThreadLocal<SimpleDateFormat> SDF = 
    ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));

// 每个线程有自己的 SimpleDateFormat 实例，互不干扰
String dateStr = SDF.get().format(new Date());
```

### 线程池中使用注意事项

```java
ExecutorService pool = Executors.newFixedThreadPool(4);

pool.submit(() -> {
    try {
        USER_CONTEXT.set("user-1");
        // 业务逻辑...
        System.out.println(USER_CONTEXT.get());
    } finally {
        USER_CONTEXT.remove();  // ⚠️ 线程池中必须 remove！
    }
});
```

**不 remove 的后果**：线程被复用后，下一个任务 `get()` 会拿到上一个任务残留的值。


# ThreadLocal最佳实践

## 核心结论

`ThreadLocal` 的最佳实践可总结为四点：**声明为 static final、使用后必须 remove()、try-finally 保护、线程池场景尤其注意**。

## 深度解析

### 实践一：声明为 private static final

```java
// ✅ 正确：static 保证只有一个 ThreadLocal 实例，final 防止被修改
private static final ThreadLocal<UserContext> USER_HOLDER = 
    ThreadLocal.withInitial(() -> null);

// ❌ 错误：每次创建新实例，导致 ThreadLocalMap 中产生大量无用的 Entry
private ThreadLocal<UserContext> userHolder = new ThreadLocal<>();
```

**原因**：每个 `ThreadLocal` 实例对应 `ThreadLocalMap` 中的一个 key，非 static 的 ThreadLocal 每次创建新对象，Map 中会积累大量废弃 key。

### 实践二：使用后必须 remove()

```java
// ✅ 标准使用模式
try {
    USER_HOLDER.set(user);
    doSomething();  // 业务逻辑
} finally {
    USER_HOLDER.remove();  // 无论成功或异常，都清理
}
```

### 实践三：封装为工具类

```java
public class UserContextHolder {
    private static final ThreadLocal<User> CONTEXT = new ThreadLocal<>();

    public static void set(User user) {
        CONTEXT.set(user);
    }

    public static User get() {
        return CONTEXT.get();
    }

    public static void clear() {
        CONTEXT.remove();
    }
}
```

### 实践四：Web 框架集成

```java
// Spring MVC Interceptor
@Component
public class UserContextInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest request, 
                             HttpServletResponse response, Object handler) {
        String token = request.getHeader("Authorization");
        User user = parseToken(token);
        UserContextHolder.set(user);
        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request, 
                                HttpServletResponse response, 
                                Object handler, Exception ex) {
        UserContextHolder.clear();  // 请求结束，必须清理
    }
}
```

### 线程池场景的完整处理

```java
ExecutorService pool = Executors.newFixedThreadPool(4);

// 方式一：在 Runnable 中清理
pool.submit(() -> {
    try {
        UserContextHolder.set(currentUser);
        doBusiness();
    } finally {
        UserContextHolder.clear();
    }
});

// 方式二：装饰器模式（推荐，避免遗漏）
public class ContextAwareRunnable implements Runnable {
    private final Runnable delegate;
    private final User user;

    public ContextAwareRunnable(Runnable delegate, User user) {
        this.delegate = delegate;
        this.user = user;
    }

    @Override
    public void run() {
        try {
            UserContextHolder.set(user);
            delegate.run();
        } finally {
            UserContextHolder.clear();
        }
    }
}
```

### 常见错误

|错误|后果|正确做法|
|---|---|---|
|非 static 声明|Map 中积累多个 key|`private static final`|
|不调用 remove()|内存泄漏 + 数据串扰|try-finally 中 remove()|
|线程池中不清理|下个任务拿到残留值|submit 前后 set/remove|
|初始化为 null 不判断|NPE|`withInitial()` 或 get()判空|
## 关联知识点

