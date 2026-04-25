---
title: 异常链面试题
tags:
  - Java/异常处理
  - 原理型
  - 问答
module: 03_异常处理
created: 2026-04-18
---

# 异常链

## Q1：什么是异常链？为什么要用异常链？

**A：**
**异常链**是指捕获一个异常后，将其作为 `cause` 包装到新异常中一起抛出，形成层层关联的异常传播路径。
使用异常链的原因：
1. **保留原始信息**：不会丢失底层异常的堆栈和描述
2. **分层解耦**：上层不暴露底层实现细节（SQL/网络异常），但保留调试信息
3. **便于排查**：日志中可以看到完整的调用链路

---

## Q2：如何实现异常链？

**A：**
```java
// 推荐方式：构造方法传入 cause
throw new ServiceException("服务层描述", originalException);

// 也可以：initCause()（只能调一次）
RuntimeException e = new RuntimeException("描述");
e.initCause(originalException);
throw e;
```

关键是自定义异常要提供 `(String message, Throwable cause)` 构造方法：
```java
public class BusinessException extends RuntimeException {
    public BusinessException(String message, Throwable cause) {
        super(message, cause);  // 调用父类，传入 cause
    }
}
```

---

## Q3：getCause() 和 getSuppressed() 有什么区别？

**A：**

| | `getCause()` | `getSuppressed()` |
|--|-------------|------------------|
| 含义 | 引发当前异常的原始原因 | 被抑制的异常（try-with-resources） |
| 数量 | 1 个（或 null） | 0 到多个 |
| 场景 | 异常链包装 | try-with-resources 关闭时抛出的异常 |

---

## Q4：catch 捕获异常后直接 throw e 和包装后 throw 有什么区别？

**A：**
- `throw e`：重新抛出原异常，保留原始堆栈信息，适合不需要转型时
- `throw new XxxException("描述", e)`：包装为新异常，添加上下文信息，适合分层转换

**注意陷阱**：
```java
catch (Exception e) {
    throw new RuntimeException(e.getMessage());  // ❌ 丢失了 cause 和堆栈！
    throw new RuntimeException("描述", e);        // ✅ 保留了完整链路
}
```

---

## Q5：如何打印完整的异常链信息？

**A：**
`e.printStackTrace()` 会打印完整的异常链，包含所有 `caused by` 信息。
也可以用日志框架：
```java
log.error("操作失败", e);  // Slf4j 会自动展开异常链
```

遍历异常链：
```java
Throwable t = e;
while (t != null) {
    System.out.println(t.getClass().getName() + ": " + t.getMessage());
    t = t.getCause();
}
```


## Q1：什么是异常链？为什么要保留原始异常信息？

**A：**

### 定义

异常链（Exception Chaining）是指将一个异常包装到另一个异常中，形成 **"原因→结果"的链条关系**。通过 `Throwable.getCause()` 方法可以沿着这条链追溯到最原始的异常（根因）。

### 为什么需要异常链？

```java
// ===== 没有 异常链 的场景 — 信息丢失！=====

public void saveOrder(Order order) {
    try {
        orderRepository.save(order);   // 这里可能抛出 SQLIntegrityConstraintViolationException
    } catch (SQLException e) {
        // ❌ 抛出新的业务异常时丢弃了原始异常
        throw new BizException("订单保存失败");
        // 调用方只能看到 "订单保存失败"，不知道是因为：
        // - 唯一索引冲突？
        // - 字段超长？
        // - 连接超时？
        // - 死锁？
        // 排查问题时完全无从下手
    }
}


// ===== 有 异常链 的场景 — 完整保留根因信息 =====

public void saveOrder(Order order) {
    try {
        orderRepository.save(order);
    } catch (SQLException e) {
        // ✅ 将原始 SQLException 作为 cause 传递
        throw new BizException("订单保存失败", e);  // e 就是 cause
        // 调用方可以通过 getCause() 找到原始的 SQLException
        // 堆栈信息完整：BizException → ... → SQLException → ... → 根因位置
    }
}
```

### 异常链示意图

```
调用链: Controller → Service → Dao → JDBC Driver

没有异常链时的堆栈：
com.example.BizException: 订单保存失败
    at com.example.OrderService.saveOrder(OrderService.java:42)
    at com.example.OrderController.createOrder(OrderController.java:28)
    → 只能看到 saveOrder 这一层的信息，丢失了底层真实原因！

有异常链时的堆栈：
com.example.BizException: 订单保存失败
    at com.example.OrderService.saveOrder(OrderService.java:42)
    at com.example.OrderController.createOrder(OrderController.java:28)
Caused by: java.sql.SQLIntegrityConstraintViolationException:
    Duplicate entry 'ORDER-20260425-001' for key 'uk_order_no'
    at mysql.jdbc.SQLError.createSQLException(SQLError.java:957)
    at mysql.jdbc.MysqlIO.checkErrorPacket(MysqlIO.java:3976)
    at mysql.jdbc.MysqlIO.sendCommand(MysqlIO.java:2525)
    ...
Caused by: java.io.IOException: Connection reset by peer
    at sun.nio.ch.SocketDispatcher.read0(Native Method)
    ...
    → 完整的因果链条一目了然！
```

### 异常链的核心价值

| **价值点** | **说明** |
|-----------|---------|
| **根因定位** | 直接找到最初出错的位置和原因 |
| **上下文丰富** | 每层都可以添加该层的语义化描述 |
| **调试效率** | 无需复现问题，堆栈直接告诉你答案 |
| **日志完整** | 生产环境排查的救命稻草 |
| **监控告警** | 可基于根因类型做分类告警策略 |

---

## Q2：异常链的三种实现方式？

**A：**

### 方式一：构造器传入 cause（推荐 ★★★）

最常用、最简洁的方式——在构造自定义异常时传入原始异常：

```java
public class ChainDemo {

    static class AppException extends RuntimeException {
        public AppException(String message, Throwable cause) {
            super(message, cause);  // ⭐ 调用父类带cause的构造器
        }
        public AppException(String message) {
            super(message);
        }
    }

    public static void main(String[] args) {
        try {
            level3();
        } catch (AppException e) {
            // 打印完整的异常链
            printChain(e);
        }
    }

    static void level3() {
        try {
            // 最底层的原始异常
            Integer.parseInt("not_a_number");  // NumberFormatException
        } catch (NumberFormatException e) {
            throw new AppException("level3 解析数据失败", e);  // 包装为AppException
        }
    }

    static void level2() {
        try {
            level3();
        } catch (AppException e) {
            throw new AppException("level2 处理数据失败", e);  // 继续向上传递
        }
    }

    static void level1() {
        try {
            level2();
        } catch (AppException e) {
            throw new AppException("level1 业务处理失败", e);  // 最终抛出
        }
    }

    static void printChain(Throwable t) {
        int level = 0;
        Throwable current = t;

        System.out.println("\n=== 异常链追踪 ===\n");

        while (current != null) {
            System.out.printf("[%d] %s%n", level++, current.getClass().getName());
            System.out.printf("    Message: %s%n", current.getMessage());

            if (current.getCause() == current) break;  // 防止自身引用环
            current = current.getCause();
        }

        /*
        输出：
        === 异常链追踪 ===

        [0] com.example.ChainDemo$AppException
            Message: level1 业务处理失败
        [1] com.example.ChainDemo$AppException
            Message: level2 处理数据失败
        [2] com.example.ChainDemo$AppException
            Message: level3 解析数据失败
        [3] java.lang.NumberFormatException
            Message: For input string: "not_a_number"
         */
    }
}
```

### 方式二：initCause() 方法

当无法在构造器中传入 cause 时（例如先创建异常对象后才知道 cause），可以使用 `initCause()`：

```java
public class InitCauseDemo {

    static class LazyException extends RuntimeException {
        public LazyException(String message) {
            super(message);  // 注意：只传了message，没传cause
            // 此时 cause = this（默认值）
        }
    }

    public static void demo() {
        LazyException lazyEx = new LazyException("操作失败");

        try {
            riskyCode();  // 可能在这里才知道具体错误
        } catch (IOException e) {
            lazyEx.initCause(e);  // ⭐ 后续设置 cause
            throw lazyEx;          // 然后才抛出
        }
    }

    private static void riskyCode() throws IOException {
        throw new IOException("磁盘写入失败");
    }
}

// 注意事项：
// 1. initCause() 只能调用一次！再次调用会抛 IllegalStateException
// 2. 如果构造器已经传入了 cause，再调 initCause 也会报错
// 3. 通常情况下优先使用方式一（构造器传入），initCause 是备用方案
```

### 方式三：Java 7+ 多异常捕获中的隐式链

```java
// Java 7 引入的多 catch 块不会自动建立异常链
try {
    // 可能抛出 FileNotFoundException 或 EOFException
} catch (FileNotFoundException | EOFException e) {
    // e 就是实际的异常对象，不存在额外的包装
    throw new BizException("文件读取失败", e);  // 如果需要包装，手动建链
}

// 但 try-with-resources 中有特殊的 Suppressed Exception（被压制的异常）：
static void suppressedExample() throws Exception {
    try (
        AutoCloseable r1 = () -> { throw new RuntimeException("r1 close"); };
        AutoCloseable r2 = () -> { throw new RuntimeException("r2 close"); };
    ) {
        throw new RuntimeException("try block");  // 主异常
    }
    // 最终抛出的是 "try block"
    // 但通过 getSuppressed() 可以获取 ["r1 close", "r2 close"] 两个被压制异常
    // 这些不是传统的 cause 链，而是并行的"附属"异常
}
```

### 三种方式对比

| **维度** | **构造器传入 cause** | **initCause()** | **多catch/Suppressed** |
|----------|---------------------|-----------------|----------------------|
| **使用频率** | ★★★★★ 最常用 | ★★☆☆☆ 少用 | 特定场景 |
| **代码简洁性** | 高（一行搞定） | 低（需要额外方法调用） | N/A |
| **灵活性** | 创建时必须知道cause | 可以延迟设置cause | 自动关联 |
| **限制** | 无 | 只能调用一次 | 受语法结构限制 |
| **推荐度** | ✅ 首选 | ⚠️ 仅特殊场景 | 了解即可 |

---

## Q3：在不同层次抛出不同异常时如何正确串联异常链？

**A：**

### 分层架构中的异常链最佳实践

```
┌─────────────┐
│ Controller  │ ← 接收请求，参数校验
│  (Web层)     │   抛出: ValidationException / 参数相关异常
├─────────────┤
│ Service     │ ← 业务逻辑编排
│  (业务层)     │   抛出: BizException (业务语义明确)
│              │   cause = 下层异常
├─────────────┤
│ Repository/ │ ← 数据访问
│  DAO(数据层)  │   抛出: DataAccessException (Spring)
│              │   cause = SQLException / RedisException 等
├─────────────┤
│ 第三方库/JDK│ ← 底层异常
│             │   SQLException, IOException, TimeoutException...
└─────────────┘
```

### 完整示例

```java
// ===== 数据层 (DAO) =====
@Repository
public class OrderDaoImpl implements OrderDao {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    @Override
    public Order findById(Long id) {
        String sql = "SELECT * FROM orders WHERE id = ?";
        try {
            return jdbcTemplate.queryForObject(sql,
                    new BeanPropertyRowMapper<>(Order.class), id);
        } catch (EmptyResultDataAccessException e) {
            // Spring JdbcTemplate 会将 SQLException 转换为 DataAccessException
            // 我们进一步转换为领域相关的异常
            throw new DataRetrievalException("查询订单失败[id=" + id + "]", e);
            // e (EmptyResultDataAccessException) 作为 cause 保留
        } catch (BadSqlGrammarException e) {
            // SQL语法错误
            throw new DataRetrievalException("SQL执行异常", e);
        }
    }

    @Override
    public void insert(Order order) {
        String sql = "INSERT INTO orders(order_no, user_id, amount) VALUES(?, ?, ?)";
        try {
            jdbcTemplate.update(sql, order.getOrderNo(), order.getUserId(), order.getAmount());
        } catch (DuplicateKeyException e) {
            // 唯一键冲突 → 转换为更有业务含义的异常
            throw new DuplicateRecordException(
                    "订单号重复[orderNo=" + order.getOrderNo() + "]", e);
        }
    }
}


// ===== 业务层 (Service) =====
@Service
public class OrderServiceImpl implements OrderService {

    @Autowired
    private OrderDao orderDao;
    @Autowired
    private StockService stockService;
    @Autowired
    private PaymentService paymentService;

    @Transactional
    public Order createOrder(CreateOrderRequest request) {
        // 1. 校验
        validateRequest(request);

        // 2. 锁库存
        try {
            stockService.lockStock(request.getItems());
        } catch (StockLockException e) {
            // 将库存异常包装为订单层面的业务异常
            throw new BizException("ORDER_STOCK_ERR",
                    "下单失败：库存不足",  // 当前层的语义
                    e);                   // 保留原始的 StockLockException
        }

        // 3. 创建订单记录
        Order order = buildOrder(request);
        try {
            orderDao.insert(order);
        } catch (DuplicateRecordException e) {
            // 订单号重复 → 可能是重试导致的幂等性问题
            log.warn("订单号已存在，尝试查询已有订单: {}", request.getOrderNo());
            return orderDao.findByOrderNo(request.getOrderNo())
                    .orElseThrow(() -> new BizException("ORDER_NOT_FOUND",
                            "订单号存在但查不到", e));
        } catch (DataRetrievalException e) {
            throw new SysException("订单持久化失败", e);  // 系统级异常
        }

        // 4. 发起支付
        try {
            PaymentResult result = paymentService.pay(order.getId(), request.getPayChannel());
            if (!result.isSuccess()) {
                throw new BizException("PAYMENT_FAILED",
                        "支付渠道[" + request.getPayChannel() + "]扣款失败",
                        result.getException());  // 支付渠道返回的异常作为cause
            }
        } catch (PaymentTimeoutException e) {
            throw new BizException("PAYMENT_TIMEOUT",
                    "支付超时，请稍后重试", e);
        } catch (PaymentChannelException e) {
            // 支付渠道不可用 → 更高一级的系统异常
            throw new ThirdPartyServiceException("PAYMENT_CHANNEL_DOWN",
                    "支付渠道暂时不可用", e, 502);
        }

        return order;
    }


    // ===== 全局异常处理器中查看异常链 =====
    @ExceptionHandler(BizException.class)
    public ResponseEntity<ApiResult> handleBiz(BizException e) {
        // 打印异常链
        Throwable rootCause = getRootCause(e);

        log.warn("业务异常[{}] | msg={} | rootCause={}[{}]",
                e.getErrorCode(),
                e.getMessage(),
                rootCause.getClass().getSimpleName(),
                rootCause.getMessage());

        return ResponseEntity.badRequest()
                .body(ApiResult.fail(e.getErrorCode(), e.getMessage()));
    }

    /**
     * 递归获取异常链的根因
     */
    private Throwable getRootCause(Throwable t) {
        Throwable cause;
        Throwable result = t;
        while ((cause = result.getCause()) != null && cause != result) {
            result = cause;
        }
        return result;
    }
}


// ===== 日志输出效果 =====
/*
WARN  [http-nio-exec-1] c.e.GlobalExceptionHandler  : 业务异常[PAYMENT_FAILED]
| msg=支付渠道[ALIPAY]扣款失败
| rootCause=AlipayBizError[ACQ.INVALID_PARAMETER]

完整异常链（ERROR级别打印完整堆栈）:
c.e.exception.BizException: 支付渠道[ALIPAY]扣款失败
    at c.e.service.OrderServiceImpl.createOrder(OrderServiceImpl.java:89)
    at c.e.controller.OrderController.create(OrderController.java:35)
Caused by: c.e.payment.PaymentFailedException: 支付接口返回失败
    at c.e.payment.AlipayClient.execute(AlipayClient.java:156)
    at c.e.payment.PaymentServiceImpl.pay(PaymentServiceImpl.java:78)
Caused by: com.alipay.api.AlipayBizException: ACQ.INVALID_PARAMETER
    at com.alipay.DefaultAlipayClient._execute(DefaultAlipayClient.java:342)
    at com.alipay.DefaultAlipayClient.execute(DefaultAlipayClient.java:201)
    → 三层清晰可见：BizException → PaymentFailedException → AlipayBizException
*/
```

### 关键原则总结

```
1. 每层只抛出本层的异常类型（不跨层暴露内部实现）
   DAO层不抛BizException；Controller层不抛SQLException

2. 每次包装都传递原始异常作为cause
   throw new XxxException("当前层描述", originalException)

3. 异常消息要包含足够的上下文
   "订单保存失败" < "订单保存失败[orderNo=ORD-001, userId=U10086]"
   后者在不需要看代码的情况下就能定位问题

4. 全局处理器统一输出异常链
   WARN/BIZ → 业务异常只打简要信息
   ERROR/SYS → 系统异常打印完整堆栈（含异常链）
```

---

## Q4：异常链中 cause 循环会导致什么问题？（cause cycle detection）

**A：**

### 什么是 Cause Cycle？

如果异常 A 的 cause 是 B，B 的 cause 又是 A（或最终指回 A），就形成了**循环引用**。JDK 对此有内置保护机制。

### JDK 的保护机制

```java
public class CycleDetectionDemo {

    public static void main(String[] args) {
        // ===== 场景1：自引用 =====
        Exception a = new Exception("A");
        try {
            a.initCause(a);  // 让 A 的 cause 指向自己
        } catch (IllegalStateException e) {
            System.out.println("捕获自引用: " + e.getMessage());
            // 输出: Self-causation not permitted
        }

        // ===== 场景2：双向引用 =====
        Exception b = new Exception("B");
        Exception c = new Exception("C");
        b.initCause(c);
        try {
            c.initCause(b);  // b → c → b 形成环
        } catch (IllegalStateException e) {
            System.out.println("捕获环形引用: " + e.getMessage());
            // 输出: Caused cycle detected
        }

        // ===== 场景3：更长的环 =====
        Exception x1 = new Exception("X1");
        Exception x2 = new Exception("X2");
        Exception x3 = new Exception("X3");
        x1.initCause(x2);
        x2.initCause(x3);
        try {
            x3.initCause(x1);  // x1 → x2 → x3 → x1 环形
        } catch (IllegalStateException e) {
            System.out.println("捕获长环: " + e.getMessage());
            // 输出: Caused cycle detected
        }
    }
}
```

### 源码层面分析

```java
// java.lang.Throwable.initCause() 源码简化版：
public synchronized Throwable initCause(Throwable cause) {
    if (this.cause != this)  // 如果已经设过cause且不是默认值(this)
        throw new IllegalStateException("Can't overwrite cause");

    // ⭐ 环检测逻辑
    if (cause == this)
        throw new IllegalArgumentException("Self-causation not permitted");

    // 向下遍历检查是否形成环
    Throwable p = cause;
    while (p != null) {
        p = p.cause;
        if (p == this)
            throw new IllegalStateException("Caused cycle detected");
    }

    this.cause = cause;  // 通过检测后才设置
    return this;
}
```

### 为什么需要防止循环？

```java
// 如果允许异常链成环，以下场景会出问题：

// 1. 打印堆栈时死循环
//    printStackTrace() 会沿着 cause 遍历，遇到环则无限递归

// 2. 序列化时无限递归
//    ObjectOutputStream 写入异常对象时会序列化整个 cause 链
//    成环会导致 StackOverflowError

// 3. 工具类遍历异常链时死循环
//    如 ExceptionUtils.getRootCause() 使用 while 循环找根因
//   遇到环则永远找不到 null 终止条件
```

### 实际开发中的注意事项

虽然 JDK 保护了显式设置的 `initCause`，但有些间接方式可能绕过检测（如反射修改私有字段）。因此在编写自己的异常工具类时也要注意防护：

```java
/**
 * 安全的异常链遍历工具 —— 自带环路保护
 */
public final class SafeChainUtils {

    /** 最大遍历深度，防止意外环路 */
    private static final int MAX_CHAIN_DEPTH = 100;

    /**
     * 安全地获取根因（带深度保护）
     */
    public static Throwable safeGetRootCause(Throwable t) {
        if (t == null) return null;

        Throwable current = t;
        Set<Throwable> visited = new HashSet<>();  // 用Set记录已访问过的节点
        int depth = 0;

        while (current != null && depth < MAX_CHAIN_DEPTH) {
            if (!visited.add(current)) {
                // 已经访问过这个异常 → 发现环路
                System.err.println("警告：检测到异常链环路！" + current.getClass().getName());
                return current;  // 返回环路起点作为"伪根因"
            }

            Throwable next = current.getCause();
            if (next == current) break;  // 自引用终止
            current = next;
            depth++;
        }

        return current != null ? current : t;
    }

    /**
     * 获取完整的异常链列表（安全版）
     */
    public static List<Throwable> safeGetThrowableList(Throwable t) {
        List<Throwable> list = new ArrayList<>();
        Set<Throwable> visited = new HashSet<>();

        Throwable current = t;
        while (current != null) {
            if (!visited.add(current)) break;  // 环路保护
            list.add(current);

            Throwable next = current.getCause();
            if (next == current) break;
            current = next;
        }

        return list;
    }

    private SafeChainUtils() {}  // 工具类禁止实例化
}
```

---

## Q5：Spring 框架中如何使用 ExceptionUtils.getRootCause() 获取根因？

**A：**

### Spring 的 ExceptionUtils 工具类

Spring Framework 在 `org.springframework.util.ExceptionUtils` 中提供了实用的异常处理工具方法。

### 核心API

```java
import org.springframework.util.ExceptionUtils;

public class SpringExceptionUtilDemo {

    public static void main(String[] args) {
        // 构造一个多层异常链
        Exception root = new RuntimeException("DB Connection timeout");
        Exception mid = new ServiceException("Order query failed", root);
        Exception top = new BizException("Create order failed", mid);


        // ===== 1. getRootCause() — 获取根因 =====
        Throwable rootCause = ExceptionUtils.getRootCause(top);
        System.out.println("根因: " + rootCause.getClass().getSimpleName()
                + ": " + rootCause.getMessage());
        // 输出: 根因: RuntimeException: DB Connection timeout


        // ===== 2. getMostSpecificCause() — 同 getRootCause() =====
        Throwable mostSpecific = ExceptionUtils.getMostSpecificCause(top);
        // 与 getRootCause() 功能相同


        // ===== 3. containsType() — 检查异常链中是否包含某类异常 =====
        boolean hasSqlEx = ExceptionUtils.containsType(top, SQLException.class);
        System.out.println("包含SQL异常? " + hasSqlEx);  // false

        boolean hasRuntimeEx = ExceptionUtils.containsType(top, RuntimeException.class);
        System.out.println("包含Runtime异常? " + hasRuntimeEx);  // true

        // 应用：判断异常链中是否有某种特定类型的异常
        try {
            doSomethingRisky();
        } catch (Exception e) {
            if (ExceptionUtils.containsType(e, DeadlockLoserDataAccessException.class)) {
                // 检测到死锁 → 自动重试
                retryOperation();
            }
        }


        // ===== 4. getNestedException() — 对于 NestedRuntimeException =====
        // Spring 自己的异常体系（如 NestedRuntimeException）支持 getNestedException()
        // ExceptionUtils 对此做了兼容处理


        // ===== 实战：在全局异常处理器中使用 =====
    }
}
```

### 在全局异常处理器中的应用

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.dao.*;
import org.springframework.transaction.TransactionSystemException;
import org.springframework.web.bind.annotation.ExceptionHandler;

@Slf4j
@RestControllerAdvice
public class AdvancedGlobalExceptionHandler {

    /**
     * 处理所有 Spring Data Access 异常
     * 利用 ExceptionUtils.getRootCause 获取真正的数据库异常
     */
    @ExceptionHandler(DataAccessException.class)
    public ResponseEntity<ApiResult> handleDataAccess(DataAccessException e) {
        // Spring 的 DataAccessException 本身就是异常链的中间层包装
        // 它把各种数据库驱动的异常都包装了进来
        Throwable rootCause = ExceptionUtils.getRootCause(e);

        log.error("[数据访问异常] 表面: {}, 根因: {}[{}]",
                e.getClass().getSimpleName(),
                rootCause.getClass().getSimpleName(),
                rootCause.getMessage(), e);

        // 根据根因类型返回不同的错误码
        String errorCode;
        if (rootCause instanceof SQLIntegrityConstraintViolationException) {
            errorCode = "DATA_DUPLICATE";  // 数据冲突
        } else if (rootCause instanceof CommunicationsException) {
            errorCode = "DATA_CONN_LOST";  // 连接断开
        } else if (rootCause instanceof QueryTimeoutException) {
            errorCode = "DATA_TIMEOUT";   // 查询超时
        } else {
            errorCode = "DATA_ACCESS_ERROR";  // 通用
        }

        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(ApiResult.fail(errorCode, "数据处理异常"));
    }

    /**
     * 处理事务相关异常
     */
    @ExceptionHandler(TransactionSystemException.class)
    public ResponseEntity<ApiResult> handleTransactionException(TransactionSystemException e) {
        Throwable rootCause = ExceptionUtils.getRootCause(e);

        log.error("[事务异常] 根因: {} - {}",
                rootCause.getClass().getName(), rootCause.getMessage(), e);

        // 判断根因是否是死锁
        if (ExceptionUtils.containsType(e, DeadlockLoserDataAccessException.class)) {
            return ResponseEntity.status(HttpStatus.CONFLICT)
                    .body(ApiResult.fail("TX_DEADLOCK", "数据冲突，请重试"));
        }

        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(ApiResult.fail("TX_ERROR", "事务处理失败"));
    }
}
```

### Spring 异常转换机制图

```
原生异常（各数据库驱动）:
  MySQL:     com.mysql.jdbc.exceptions.MySQLSyntaxErrorException
  PostgreSQL: org.postgresql.util.PSQLException
  Oracle:    oracle.jdbc.driver.OracleSQLException
  H2:        org.h2.jdbc.JdbcSQLSyntaxErrorException
      ↓ Spring JDBC Abstraction (JdbcTemplate 等)
  ┌───────────────────────────────────────┐
  │  org.springframework.dao.DataAccessException  │ ← 最高层抽象
  │  ├── BadSqlGrammarException               │ ← SQL语法错误
  │  ├── DuplicateKeyException               │ ← 唯一键冲突
  │  ├── DataIntegrityViolationException     │ ← 完整性约束违反
  │  ├── CannotAcquireLockException           │ ← 无法获取锁
  │  ├── DeadlockLoserDataAccessException     │ ← 死锁检测
  │  ├── QueryTimeoutException               │ ← 查询超时
  │  └── TransientDataAccessException        │ ← 临时性错误(可重试)
  └───────────────────────────────────────┘
      ↓ 每个都持有原始异常作为 cause
  ExceptionUtils.getRootCause() → 获取到底层驱动异常
```

---

## Q6：日志记录时应该记录原始异常还是包装异常？最佳实践？

**A：**

### 核心原则

> **每层只记一次日志！不要在同一异常链上重复记录！**

### 反模式：重复日志污染

```java
// ===== ❌ 错误做法：每层都打日志 =====

// DAO 层
@Override
public User findById(Long id) {
    try {
        return jdbc.queryForObject(sql, mapper, id);
    } catch (EmptyResultDataAccessException e) {
        log.error("DAO层: 查询用户失败[id=" + id + "]", e);  // 日志①
        throw new DataRetrievalException("查询失败", e);     // 传递上去
    }
}

// Service 层
public User getUser(Long id) {
    try {
        return dao.findById(id);
    } catch (DataRetrievalException e) {
        log.error("Service层: 获取用户失败[id=" + id + "]", e);  // 日志②（重复！）
        throw new BizException("USER_NOT_FOUND", "用户不存在", e);  // 继续传递
    }
}

// GlobalExceptionHandler
@ExceptionHandler(Exception.class)
public Result handle(Exception e) {
    log.error("全局: 未预期异常", e);  // 日志③（又重复！）
    return Result.fail(500, "系统错误");
}

/* 结果：同一个错误产生了 3 条几乎相同的日志！
   日志膨胀 ×3，排查时反而干扰视线。
   而且 ③号日志是最外层的，通常信息量最少。 */
```

### 正确做法：分层日志策略

```java
// ===== ✅ 正确做法：每层职责分明 =====

/**
 * 日志分层策略：
 * - DAO 层：不打日志（或仅DEBUG级别），交给上层处理
 * - Service 层：WARN 级别记录业务异常
 * - GlobalHandler：ERROR 级别记录系统异常（兜底）
 */

// DAO 层 —— 不打日志，只做异常转换
@Override
public User findById(Long id) {
    try {
        return jdbc.queryForObject(sql, mapper, id);
    } catch (EmptyResultDataAccessException e) {
        // 不打日志！只是纯异常转换
        throw new DataRetrievalException("User not found: " + id, e);
    }
}

// Service 层 —— WARN 记录业务异常（不带堆栈，节省空间）
public User getUser(Long id) {
    try {
        return dao.findById(id);
    } catch (DataRetrievalException e) {
        // ⚠️ 只打一行摘要，不要打印完整堆栈
        log.warn("用户未找到 | id={} | rootCause={}: {}",
                id,
                ExceptionUtils.getRootCause(e).getClass().getSimpleName(),
                ExceptionUtils.getRootCause(e).getMessage());
        // 不带第四个参数 e（即不打印堆栈），因为这是预期的业务分支

        throw new BizException("USER_404", "用户不存在[id=" + id + "]", e);
    }
}

// GlobalHandler —— 分级处理
@ExceptionHandler(BizException.class)  // 业务异常
public Result handleBiz(BizException e) {
    // 不打日志！Service 层已经打过了
    return Result.fail(e.getCode(), e.getMessage());  // 400
}

@ExceptionHandler(Exception.class)   // 未预期异常（唯一打 ERROR 的地方）
public Result handleUnexpected(Exception e) {
    // ⭐ 只有这里才打印完整的堆栈跟踪（含异常链）
    log.error("未预期异常 | type={} | msg={}",
            e.getClass().getName(), e.getMessage(), e);
    // 第四个参数 e 触发完整堆栈输出

    return Result.fail(500, isProd() ? "系统繁忙" : e.getMessage());
}
```

### 日志决策矩阵

| **异常类型** | **日志级别** | **是否打堆栈** | **在哪里打** |
|------------|-----------|--------------|-----------|
| 业务异常 (BizException) | **WARN** | ❌ 不打 | Service 层 |
| 参数校验异常 | **WARN** | ❌ 不打 | Handler 层 |
| 可重试的临时异常 | **INFO/WARN** | 可选 | 重试组件内 |
| 未预期系统异常 | **ERROR** | **✅ 必须打** | GlobalHandler |
| 致命错误（OOM等） | **ERROR** | **✅ 必须打** | 全局兜底 |

### 日志格式建议

```java
// 推荐的异常日志格式（结构化，便于ELK解析）:

log.warn("{} | code={} | ctx={} | rootCause={}[{}]",
        operationName,          // 操作名: "create_order"
        exception.getErrorCode(), // 错误码: "STOCK_001"
        exception.getContext(),  // 上下文: {productId=P001, qty=5}
        rootCauseName,          // 根因类名: "DataIntegrityViolationException"
        rootCauseMessage        // 根因消息: "Duplicate entry..."
        // 注意：不加第6个exception参数 → 不输出堆栈（WARN级别）
);

log.error("{} | unexpected error", operationName, e);
// 加上 e → 输出完整堆栈和异常链（ERROR级别才需要）
```

---

## Q7：异常链与 JDK 1.4 Throwable 初始化方法 initCause() 的关系？

**A：**

### 历史背景

**异常链机制是在 JDK 1.4 中引入的**。在此之前（JDK 1.3 及之前），Java 异常不支持 cause 链，开发者只能选择：

1. 丢弃原始异常（信息丢失）
2. 把原始异常的消息拼接到新异常中（字符串拼接，丢失类型信息）

JDK 1.4 为 `Throwable` 类新增了三个关键成员：

```java
// JDK 1.4 新增的内容：
public class Throwable {

    // 新增字段：存储 cause
    private Throwable cause = this;  // 默认指向自身

    // 新增构造器：支持传入 cause
    protected Throwable(String message, Throwable cause) { ... }

    // 新增方法：显式设置 cause
    public synchronized Throwable initCause(Throwable cause) { ... }

    // 新增方法：获取 cause
    public Throwable getCause() { return cause; }

    // 新增方法（printStackTrace 内部使用）：获取完整异常链轨迹
    public final void printStackTrace(PrintStream s) {
        // 先打印自身的栈帧
        synchronized (s.lock()) {
            s.println(this);
            StackTraceElement[] trace = getOurStackTrace();
            for (StackTraceElement traceElement : trace)
                s.println("\tat " + traceEntity);

            // ⭐ 递归打印 cause 链
            Throwable ourCause = getCause();
            if (ourCause != null)
                ourCause.printStackTraceAsCause(s, trace);
        }
    }
}
```

### initCause() 的设计约束

```java
public class InitCauseRules {

    public static void demonstrate() {
        Exception ex = new Exception();

        // ===== 规则1：每个异常只能设置一次 cause =====
        ex.initCause(new RuntimeException("First cause"));  // OK
        // ex.initCause(new RuntimeException("Second cause"));  // IllegalStateException!

        // ===== 规则2：不能覆盖构造器已设置的 cause =====
        Exception ex2 = new Exception("msg", new RuntimeException("from constructor"));
        // ex2.initCause(new RuntimeException("override attempt"));  // IllegalStateException!

        // ===== 规则3：默认 cause 是自身（表示"无cause"）=====
        Exception ex3 = new Exception("standalone");
        System.out.println(ex3.getCause() == ex3);  // true! 默认指向自己
        // getCause() == this 表示没有外部设置的 cause

        // ===== 规则4：cause 不能为 null（但语义上等同于无cause）=====
        Exception ex4 = new Exception("test");
        ex4.initCause(null);  // 合法！null 表示清除 cause
        System.out.println(ex4.getCause());  // null

        // ===== 规则5：自引用被禁止 =====
        Exception ex5 = new Exception("self");
        // ex5.initCause(ex5);  // IllegalArgumentException!

        // ===== 实际应用场景：何时必须用 initCause？ =====
        //
        // 当你的异常类继承自 legacy 类（不能修改父类的构造器签名），
        // 或者需要在异常创建之后才能确定 cause 时：
        LegacyException legacy = new LegacyException("something wrong");
        try {
            mightFail();
        } catch (IOException e) {
            legacy.initCause(e);  // 后续附加 cause
            throw legacy;
        }
    }
}

class LegacyException extends RuntimeException {
    // 假设这是一个遗留类，只有单参构造器，无法传入 cause
    public LegacyException(String message) {
        super(message);
    }
}
```

### initCause() vs 构造器传入 cause：如何选择？

```java
// 选择指南：

// 情况A：你拥有对异常类的控制权（自己定义的异常类）
// → 推荐：提供带 cause 的构造器（方式一）

class MyException extends RuntimeException {
    public MyException(String message, Throwable cause) {
        super(message, cause);  // 最佳实践
    }
}

// 情况B：你不能修改异常类的构造器
// → 必须用 initCause()（方式二）

void workWithLegacyException() {
    ThirdPartyLibException ex = new ThirdPartyLibException("error");
    try {
        dangerousOp();
    } catch (SocketTimeoutException e) {
        ex.initCause(e);  // 唯一的选择
        throw ex;
    }
}

// 情况C：创建异常时还不知道 cause
// → 必须 initCause()

Exception deferred = new BusinessException("operation failed");
// ... 一些中间操作 ...
try {
    finalStep();
} catch (Exception e) {
    deferred.initCause(e);  // 此时才知道 cause
    throw deferred;
}
```

### JDK 版本兼容性

```java
// 如果你需要维护运行在 JDK 1.3 及更早版本的代码：
// initCause() 和 getCause() 都不存在！
// 兼容写法：

if (isJdk14OrLater()) {
    exception.initCause(rootCause);  // JDK 1.4+
} else {
    // JDK 1.3 回退方案：将 cause 信息拼接到消息中
    String enhancedMsg = exception.getMessage()
            + " [caused by: " + rootCause.getClass().getName()
            + ": " + rootCause.getMessage() + "]";
    // 丢失了 cause 类型信息和堆栈，但没有更好的办法
}
// （现在基本不需要考虑这种兼容问题了）
```
