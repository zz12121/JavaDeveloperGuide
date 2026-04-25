---
title: 自定义异常面试题
tags:
  - Java/异常处理
  - 原理型
  - 问答
module: 03_异常处理
created: 2026-04-25
---

# 自定义异常

## Q1：为什么自定义异常通常继承 RuntimeException 而不是 Exception？

**A：**

### 核心原因：避免强制调用者处理可预见的业务异常

Java 异常体系分为两大类：

```
Throwable
├── Error（不可恢复的系统级错误，不应捕获）
│   └── OutOfMemoryError, StackOverflowError...
└── Exception
    ├── Checked Exception（编译器强制必须处理）  ← 继承 Exception
    │   ├── IOException, SQLException...
    │   └── 自定义 checked 异常
    └── Unchecked Exception（运行时异常，不强制处理）← 继承 RuntimeException
        ├── NullPointerException, IllegalArgumentException...
        └── 自定义运行时异常
```

#### 继承 Exception 的问题

```java
// ❌ 如果继承 Exception（Checked Exception）
public class BusinessException extends Exception {
    public BusinessException(String message) { super(message); }
}

// 那么所有调用的地方都必须 try-catch 或 throws：
public User getUser(Long userId) throws BusinessException {  // 必须声明
    if (userId == null) {
        throw new BusinessException("用户ID不能为空");  // 调用方被迫处理
    }
    // ...
}

public void handleRequest() {
    // 每一层都要处理或者继续抛出...代码被污染
    try {
        User user = getUser(null);
        // ... 使用user
    } catch (BusinessException e) {
        // 业务异常本质上是可以预期的，不应该用 try-catch 包裹所有业务逻辑
        log.error("业务异常", e);
    } catch (SQLException e) {
        // 真正的异常被混在业务"异常"中
    }
}
```

#### 继承 RuntimeException 的优势

```java
// ✅ 推荐做法：继承 RuntimeException
public class BizException extends RuntimeException {
    private final ErrorCode errorCode;  // 错误码
    private Object[] args;              // 动态参数

    public BizException(ErrorCode code) {
        super(code.getMessage());
        this.errorCode = code;
    }

    public BizException(ErrorCode code, Throwable cause) {
        super(code.getMessage(), cause);
        this.errorCode = code;
    }

    public ErrorCode getErrorCode() { return errorCode; }
}

// 枚举定义错误码
public enum ErrorCode {
    USER_NOT_FOUND(10001, "用户不存在"),
    INVALID_PARAM(10002, "参数校验失败"),
    PERMISSION_DENIED(10003, "无权限操作");

    private final int code;
    private final String message;
    // constructor & getters...
}

// 使用时非常简洁——不需要到处 try-catch 或 throws
public User getUser(Long userId) {
    if (userId == null) {
        throw new BizException(ErrorCode.INVALID_PARAM);  // 直接抛出，无需声明
    }

    User user = userRepository.findById(userId)
            .orElseThrow(() -> new BizException(ErrorCode.USER_NOT_FOUND));
    return user;
}

// 在全局统一处理（如 Spring @RestControllerAdvice），一处处理全部异常
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(BizException.class)
    public ResponseEntity<ApiResult> handleBizException(BizException e) {
        return ResponseEntity.status(HttpStatus.BAD_REQUEST)
                .body(ApiResult.fail(e.getErrorCode().getCode(), e.getMessage()));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiResult> handleException(Exception e) {
        log.error("系统异常", e);
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(ApiResult.fail(50000, "系统繁忙"));
    }
}
```

### 对比总结

| **维度** | **继承 Exception (Checked)** | **继承 RuntimeException (Unchecked)** |
|----------|---------------------------|-------------------------------------|
| **是否强制处理** | ✅ 编译期强制 | ❌ 不强制 |
| **代码侵入性** | 高（每层都需 throws/try-catch） | 低（只在全局处理器处理） |
| **与框架集成** | 不友好（Spring MVC 不推荐） | 友好（@ExceptionHandler 完美配合） |
| **API 设计** | 方法签名被污染 | 干净整洁 |
| **阿里巴巴规范** | ❌ 禁止直接使用 | ✅ 推荐 |

---

## Q2：如何设计业务异常体系？（错误码+错误信息+异常链）

**A：**

### 分层设计原则

一个成熟的业务异常体系应该包含三个层次：

```
1. 基础异常类 — 所有业务异常的父类
2. 异常枚举 — 定义错误码和默认消息（集中管理）
3. 具体异常子类 — 可选，按领域细分（可选但推荐）
```

### 完整实现方案

```java
import lombok.Getter;

// ===== 第一层：基础异常类 =====
/**
 * 所有业务异常的基类
 * 特点：
 * 1. 继承 RuntimeException（不受检）
 * 2. 包含错误码、错误消息、原始异常（异常链）
 * 3. 支持国际化消息模板
 */
@Getter
public class BaseException extends RuntimeException {

    /** 错误码 */
    private final String errorCode;

    /** HTTP状态码（供REST接口返回时使用） */
    private final int httpStatus;

    /**
     * 带错误码和消息的构造器
     */
    public BaseException(String errorCode, String message) {
        super(message);
        this.errorCode = errorCode;
        this.httpStatus = 400;
    }

    /**
     * 带错误码、消息和原始异常的构造器（保留异常链）
     */
    public BaseException(String errorCode, String message, Throwable cause) {
        super(message, cause);
        this.errorCode = errorCode;
        this.httpStatus = 400;
    }

    /**
     * 带HTTP状态码的构造器
     */
    public BaseException(String errorCode, String message, int httpStatus) {
        super(message);
        this.errorCode = errorCode;
        this.httpStatus = httpStatus;
    }

    public BaseException(String errorCode, String message, Throwable cause, int httpStatus) {
        super(message, cause);
        this.errorCode = errorCode;
        this.httpStatus = httpStatus;
    }


    // ===== 第二层：具体异常子类（按领域划分）=====

    /** 参数校验异常（400） */
    public static class ValidationException extends BaseException {
        public ValidationException(String message) {
            super("VALIDATION_ERROR", message, 400);
        }
        public ValidationException(String field, String message) {
            super("VALIDATION_ERROR", field + ": " + message, 400);
        }
    }

    /** 业务逻辑异常（一般400） */
    public static class BizLogicException extends BaseException {
        public BizLogicException(String code, String message) {
            super(code, message, 400);
        }
        public BizLogicException(String code, String message, Throwable cause) {
            super(code, message, cause, 400);
        }
    }

    /** 资源未找到异常（404） */
    public static class NotFoundException extends BaseException {
        public NotFoundException(String resource, String id) {
            super("NOT_FOUND", resource + "[" + id + "] 不存在", 404);
        }
    }

    /** 权限不足异常（403） */
    public static class ForbiddenException extends BaseException {
        public ForbiddenException(String message) {
            super("FORBIDDEN", message, 403);
        }
    }

    /** 未登录/认证异常（401） */
    public static class UnauthorizedException extends BaseException {
        public UnauthorizedException() {
            super("UNAUTHORIZED", "未登录或token已过期", 401);
        }
    }

    /** 第三方服务调用异常（502/504等） */
    public static class ThirdPartyException extends BaseException {
        public ThirdPartyException(String service, Throwable cause) {
            super("THIRD_PARTY_ERROR", "第三方服务[" + service + "]调用失败", cause, 502);
        }
    }
}


// ===== 第三层：错误码枚举（集中管理所有业务错误码）=====
/**
 * 全局错误码定义
 *
 * 规则建议：
 * - 格式：模块号(2位) + 子模块(2位) + 序列号(3位) → 共7位数字
 * - 例如：10(用户模块) + 01(注册) + 001 → 1001001
 */
@Getter
@AllArgsConstructor
public enum ErrorCodes {

    // === 通用 (00) ===
    SUCCESS("0000000", "成功"),
    SYSTEM_ERROR("0000999", "系统内部错误"),

    // === 用户模块 (10) ===
    USER_NOT_FOUND("1001001", "用户不存在"),
    USER_DISABLED("1001002", "用户已被禁用"),
    USER_PASSWORD_WRONG("1002001", "密码错误"),
    USER_REGISTER_DUPLICATE("1003001", "该手机号已注册"),

    // === 订单模块 (20) ===
    ORDER_NOT_FOUND("2001001", "订单不存在"),
    ORDER_STATUS_ERROR("2001002", "订单状态不允许此操作"),
    ORDER_PAID_TIMEOUT("2002001", "订单支付超时"),

    // === 库存模块 (30) ===
    STOCK_INSUFFICIENT("3001001", "库存不足"),
    STOCK_LOCKED("3001002", "商品库存已被锁定"),

    // === 支付模块 (40) ===
    PAY_AMOUNT_MISMATCH("4001001", "支付金额不匹配"),
    PAY_CHANNEL_ERROR("4002001", "支付渠道不可用");

    private final String code;
    private final String defaultMessage;
}


// ===== 使用示例 =====
public class UserServiceImpl implements UserService {

    @Autowired
    private UserRepository userRepo;

    @Override
    public User login(String phone, String password) {
        // 1. 参数校验
        if (StringUtils.isBlank(phone)) {
            throw new BaseException.ValidationException("phone", "手机号不能为空");
        }

        // 2. 查询用户
        User user = userRepo.findByPhone(phone);
        if (user == null) {
            throw new BaseException.BizLogicException(
                    ErrorCodes.USER_NOT_FOUND.getCode(),
                    "手机号[" + phone + "]未注册");
        }

        // 3. 校验状态
        if (!user.isEnabled()) {
            throw new BaseException.BizLogicException(
                    ErrorCodes.USER_DISABLED.getCode(),
                    ErrorCodes.USER_DISABLED.getDefaultMessage());
        }

        // 4. 校验密码
        if (!passwordEncoder.matches(password, user.getPassword())) {
            throw new BaseException.BizLogicException(
                    ErrorCodes.USER_PASSWORD_WRONG.getCode(),
                    ErrorCodes.USER_PASSWORD_WRONG.getDefaultMessage());
        }

        return user;
    }
}
```

### 错误码设计规范

```java
/*
错误码设计最佳实践：

1. 数值型 vs 字符串型？
   - 推荐字符串型："USER_001"、"ORDER_002"
   - 优点：可读性好，便于日志检索和前后端联调
   - 缺点：稍长（但可以接受）

2. 层次化结构：
   10xxx — 用户相关
   20xxx — 订单相关
   30xxx — 商品/库存相关
   50xxx — 支付相关
   90xxx — 第三方依赖

3. 与 HTTP 状态码的关系：
   - 400 Bad Request      → 参数校验类异常
   - 401 Unauthorized     → 未认证
   - 403 Forbidden         → 无权限
   - 404 Not Found         → 资源不存在
   - 409 Conflict          → 数据冲突（并发修改等）
   - 500 Internal Server   → 未预期的系统异常
   - 502/503/504           → 第三方服务问题
*/
```

---

## Q3：自定义异常中应该包含哪些信息？（错误码、原始异常、上下文数据）

**A：**

### 最小必要信息集

一个高质量的自定义异常至少应包含以下四项信息：

| **信息** | **字段名** | **作用** | **必要性** |
|----------|-----------|---------|-----------|
| **错误码** | `errorCode` | 唯一标识异常类型，用于程序化处理 | 必须 |
| **错误消息** | `message` | 人可读的描述，便于调试 | 必须 |
| **原始异常** | `cause` (Throwable) | 保留异常链，追踪根因 | 强烈推荐 |
| **上下文数据** | `context` / `args` | 出错时的关键参数值，辅助排查 | 推荐 |
| **HTTP状态码** | `httpStatus` | REST API 返回时使用 | 推荐（Web项目） |
| **时间戳** | `timestamp` | 发生时间 | 可选（日志中已有） |

### 完整实现

```java
import java.time.LocalDateTime;
import java.util.HashMap;
import java.util.Map;

/**
 * 企业级自定义异常——包含完整上下文信息的异常类
 */
public class RichBizException extends RuntimeException {

    private final String errorCode;
    private final int httpStatus;
    private final Map<String, Object> context = new HashMap<>();  // 上下文数据
    private final LocalDateTime occurredAt;                        // 发生时间

    // ===== 构造器 =====
    public RichBizException(String errorCode, String message) {
        super(message);
        this.errorCode = errorCode;
        this.httpStatus = 500;
        this.occurredAt = LocalDateTime.now();
    }

    public RichBizException(String errorCode, String message, Throwable cause) {
        super(message, cause);
        this.errorCode = errorCode;
        this.httpStatus = 500;
        this.occurredAt = LocalDateTime.now();
    }

    // ===== Builder 风格的上下文设置 =====
    /**
     * 向异常附加上下文信息（链式调用）
     * 用法: throw new RichBizException("ERR_001", "xxx").put("userId", userId).put("orderId", orderId);
     */
    public RichBizException put(String key, Object value) {
        context.put(key, value);
        return this;  // 返回this支持链式调用
    }

    // ===== Getter =====
    public String getErrorCode() { return errorCode; }
    public int getHttpStatus() { return httpStatus; }
    public Map<String, Object> getContext() { return Collections.unmodifiableMap(context); }
    public LocalDateTime getOccurredAt() { return occurredAt; }

    // ===== 重写 toString 用于日志输出 =====
    @Override
    public String toString() {
        StringBuilder sb = new StringBuilder()
                .append("[RichBizException] ")
                .append("code=").append(errorCode)
                .append(", msg='").append(getMessage()).append("'")
                .append(", time=").append(occurredAt);

        if (!context.isEmpty()) {
            sb.append(", context=").append(context);
        }

        if (getCause() != null) {
            sb.append(", cause=").append(getCause().getClass().getSimpleName())
              .append(": ").append(getCause().getMessage());
        }

        return sb.toString();
    }
}


// ===== 实战使用示例 =====
public class OrderService {

    public Order createOrder(Long userId, List<OrderItem> items) {
        // 1. 校验用户
        User user = findUser(userId);
        if (user == null) {
            throw new RichBizException("USER_001", "用户不存在")
                    .put("userId", userId)
                    .put("operation", "CREATE_ORDER");  // 附加上下文
        }

        // 2. 校验库存
        for (OrderItem item : items) {
            Stock stock = stockRepo.findByProductId(item.getProductId());
            if (stock == null || stock.getQuantity() < item.getQuantity()) {
                throw new RichBizException("STOCK_001", "库存不足")
                        .put("productId", item.getProductId())
                        .put("requestQty", item.getQuantity())
                        .put("availableQty", stock != null ? stock.getQuantity() : 0)
                        .put("orderIdCandidate", generateOrderId());  // 有助于排查
            }
        }

        // 3. 创建订单（可能抛数据库异常）
        try {
            Order order = doCreateOrder(userId, items);
            return order;
        } catch (DataAccessException e) {
            // 保留原始异常作为 cause
            throw new RichBizException("ORDER_001", "订单创建失败", e)
                    .put("userId", userId)
                    .put("itemCount", items.size())
                    .put("totalAmount", calcTotal(items));  // 上下文帮助排查
        }
    }
}


// ===== 全局异常处理器中利用这些信息 =====
@Slf4j
@RestControllerAdvice
public class RichGlobalHandler {

    @ExceptionHandler(RichBizException.class)
    public ResponseEntity<Map<String, Object>> handleRich(RichBizException ex) {
        log.warn("业务异常: {}", ex.toString());  // 利用重写的toString输出完整信息

        Map<String, Object> result = new HashMap<>();
        result.put("code", ex.getErrorCode());
        result.put("message", ex.getMessage());
        result.put("timestamp", ex.getOccurredAt().toString());

        // 开发环境返回上下文信息，生产环境隐藏敏感信息
        if (isDevEnvironment()) {
            result.put("debugContext", ex.getContext());
        }

        return ResponseEntity.status(ex.getHttpStatus()).body(result);
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<Map<String, Object>> handleGeneric(Exception ex) {
        log.error("未预期异常", ex);  // 始终打印完整堆栈

        Map<String, Object> result = new HashMap<>();
        result.put("code", "SYS_ERROR");
        result.put("message", isDevEnvironment() ? ex.getMessage() : "系统繁忙");
        result.put("timestamp", LocalDateTime.now().toString());

        return ResponseEntity.status(500).body(result);
    }
}
```

### 为什么上下文信息如此重要？

```java
// 没有 context 的异常日志：
ERROR 2026-04-25 20:30:00 [http-nio-8080-exec-1] BizException: 库存不足

// 有 context 的异常日志：
WARN  2026-04-25 20:30:00 [http-nio-8080-exec-1]
[RichBizException] code=STOCK_001, msg='库存不足',
time=2026-04-25T20:30:00,
context={productId=P10086, requestQty=5, availableQty=2},
cause=null
// → 一眼就能看出是 P10086 商品要5个但只有2个！无需翻代码查参数。
```

---

## Q4：为什么阿里巴巴Java开发手册禁止直接继承 Exception 而推荐 RuntimeException？

**A：**

### 阿里巴巴规范原文

> 【强制】类库中定义通过异常来反馈的错误，不要使用 errno，而异常也不要用来做流程控制/条件控制。
> 【强制】定义两种异常体系：BizException（业务异常，继承 RuntimeException）、SystemException（系统异常，也继承 RuntimeException）。
> 【强制】不要在 finally 块中使用 return / throw。finally 块中的 return 会覆盖 try 块中的返回。

### 核心理由分析

#### 理由1：Checked Exception 的"传染性"污染代码

```java
// 当一个底层方法抛出Checked Exception后，它会沿着调用链逐层传播：
// Dao层:
public User findById(Long id) throws DataAccessException { ... }

// Service层（被迫throws）:
public User getUser(Long id) throws DataAccessException {  // ← 被迫签名
    return dao.findById(id);
}

// Controller层（继续throws或try-catch）:
public ResponseEntity<User> getUser(@PathVariable Long id) throws DataAccessException {  // ← 继续污染
    return ResponseEntity.ok(service.getUser(id));
}
// 最终到框架入口处（Spring DispatcherServlet）才统一处理
// 但中间每一层的签名都被污染了
```

#### 理由2：与 Spring 框架的设计理念冲突

```java
// Spring 框架的核心设计哲学之一就是减少 Checked Exception 的使用：
// - @Transactional 默认只回滚 RuntimeException 和 Error，不回滚 Checked Exception
// - @RestControllerAdvice 的 @ExceptionHandler 天然适合处理 Unchecked Exception
// - Spring Template (JdbcTemplate 等) 将所有 Checked Exception 包装为 RuntimeException (DataAccessException)

@Transactional
public void transfer(Account from, Account to, BigDecimal amount) {
    from.debit(amount);
    to.credit(amount);
    // 如果这里抛的是 Checked Exception (extends Exception)，事务不会回滚！
    // 只有 RuntimeException 才会触发回滚！
}
```

#### 理由3：函数式编程不兼容 Checked Exception

```java
// Java 8 Lambda 表达式不支持抛出 Checked Exception：
List<User> users = userIds.stream()
    .map(id -> userService.findById(id))  // findById 如果 throws Exception → 编译报错!
    .collect(Collectors.toList());

// 解决方案很丑陋：必须在lambda内try-catch
List<User> users = userIds.stream()
    .map(id -> {
        try {
            return userService.findById(id);
        } catch (Exception e) {
            throw new RuntimeException(e);  // 包装为RuntimeException
        }
    })
    .collect(Collectors.toList());
// 既然最终还是要包装成 RuntimeException，为什么不一开始就用它呢？
```

#### 理由4：实际生产中的经验教训

```
大型项目中 Checked Exception 导致的问题：

1. 吞异常现象泛滥
   try { ... } catch (CheckedException e) { /* ignore */ }  // 太常见了

2. 无意义的 throws 链
   public void a() throws E1 { b(); }
   public void b() throws E1 { c(); }
   public void c() throws E1 { d(); }
   public void d() throws E1 { ... }  // 每一层都是无脑传递

3. 异常处理策略无法统一
   有的地方log，有的地方ignore，有的地方包装再抛
   因为没有全局统一的入口点
```

### 正确的自定义异常实践

```java
/**
 * 阿里巴巴推荐的异常体系设计
 */

// 1. 基础运行时异常
public abstract class AppBaseException extends RuntimeException {
    protected AppBaseException(String message) { super(message); }
    protected AppBaseException(String message, Throwable cause) { super(message, cause); }
    public abstract String getErrCode();  // 子类实现
}

// 2. 业务异常（客户端错误 4xx）
public final class BizException extends AppBaseException {
    private final String errCode;

    public BizException(String errCode, String message) {
        super(message);
        this.errCode = errCode;
    }

    public BizException(String errCode, String message, Throwable cause) {
        super(message, cause);
        this.errCode = errCode;
    }

    @Override public String getErrCode() { return errCode; }
}

// 3. 系统异常（服务端错误 5xx）
public final class SysException extends AppBaseException {
    public SysException(String message) { super(message); }
    public SysException(String message, Throwable cause) { super(message, cause); }
    @Override public String getErrCode() { return "SYSTEM_ERROR"; }
}

// 4. 全局统一处理
@Slf4j
@RestControllerAdvice
public class UnifiedExceptionHandler {

    @ExceptionHandler(BizException.class)
    public Result<?> handleBiz(BizException e) {
        log.info("业务异常[{}]: {}", e.getErrCode(), e.getMessage());
        return Result.fail(e.getErrCode(), e.getMessage());  // 400
    }

    @ExceptionHandler(SysException.class)
    public Result<?> handleSys(SysException e) {
        log.error("系统异常", e);
        return Result.fail("SYSTEM_ERROR", "系统繁忙，请稍后再试");  // 500
    }

    @ExceptionHandler(Throwable.class)
    public Result<?> handleUnknown(Throwable t) {
        log.error("未知异常", t);
        return Result.fail("UNKNOWN", "系统繁忙");  // 500
    }
}
```

---

## Q5：如何在全局异常处理器中统一处理自定义异常？

**A：**

### Spring Boot 全局异常处理器完整实现

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.http.HttpStatus;
import org.springframework.validation.BindException;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.ResponseStatus;
import org.springframework.web.bind.annotation.RestControllerAdvice;
import javax.validation.ConstraintViolation;
import javax.validation.ConstraintViolationException;

/**
 * 全局异常处理器 —— 统一处理所有异常并返回标准格式响应
 */
@Slf4j
@RestControllerAdvice  // = @ControllerAdvice + @ResponseBody
public class GlobalExceptionAdvisor {

    // ===== 1. 业务异常处理 =====
    @ExceptionHandler(BizException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)  // 400
    public ApiResult<Void> handleBusinessException(BizException e) {
        log.warn("[业务异常] code={}, msg={}, ctx={}",
                e.getErrorCode(), e.getMessage(), e.getContext());

        return ApiResult.<Void>builder()
                .code(Integer.parseInt(e.getErrorCode()))
                .msg(e.getMessage())
                .timestamp(System.currentTimeMillis())
                .build();
    }

    // ===== 2. 参数校验异常 (@Validated/@Valid 失败) =====
    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)  // 400
    public ApiResult<Void> handleValidationException(MethodArgumentNotValidException e) {
        String errorMsg = e.getBindingResult().getFieldErrors().stream()
                .map(fe -> fe.getField() + ": " + fe.getDefaultMessage())
                .reduce((a, b) -> a + "; " + b)
                .orElse("参数校验失败");

        log.warn("[参数校验异常] {}", errorMsg);

        return ApiResult.<Void>builder()
                .code(400)
                .msg(errorMsg)
                .build();
    }

    // ===== 3. 约束违反异常 (@NotNull/@NotBlank 等直接标注在参数上) =====
    @ExceptionHandler(ConstraintViolationException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ApiResult<Void> handleConstraintViolation(ConstraintViolationException e) {
        String errorMsg = e.getConstraintViolations().stream()
                .map(ConstraintViolation::getMessage)
                .reduce((a, b) -> a + "; " + b)
                .orElse("约束违反");

        return ApiResult.<Void>builder().code(400).msg(errorMsg).build();
    }

    // ===== 4. 资源未找到 =====
    @ExceptionHandler(ResourceNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)  // 404
    public ApiResult<Void> handleNotFound(ResourceNotFoundException e) {
        log.warn("[资源未找到] {}", e.getMessage());
        return ApiResult.<Void>builder().code(404).msg(e.getMessage()).build();
    }

    // ===== 5. 权限不足 =====
    @ExceptionHandler(AccessDeniedException.class)
    @ResponseStatus(HttpStatus.FORBIDDEN)  // 403
    public ApiResult<Void> handleAccessDenied(AccessDeniedException e) {
        log.warn("[权限不足] {}", e.getMessage());
        return ApiResult.<Void>builder().code(403).msg("无权访问").build();
    }

    // ===== 6. 第三方服务异常 =====
    @ExceptionHandler(ThirdPartyException.class)
    public ApiResult<Void> handleThirdParty(ThirdPartyException e) {
        log.error("[第三方服务异常] service={}, cause={}",
                e.getServiceName(), e.getCause() != null ? e.getCause().getMessage() : "N/A",
                e);

        // 生产环境不暴露内部错误细节
        return ApiResult.<Void>builder()
                .code(e.getHttpStatus())
                .msg(isProd() ? "外部服务暂时不可用" : e.getMessage())
                .build();
    }

    // ===== 7. 兜底：处理所有未被上面的handler处理的异常 =====
    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)  // 500
    public ApiResult<Void> handleGeneralException(Exception e) {
        log.error("[未预期异常]", e);  // ⭐ 这里一定要打 ERROR 日志！

        return ApiResult.<Void>builder()
                .code(500)
                .msg(isProd() ? "服务器内部错误" : e.getMessage())
                .build();
    }
}


// ===== 统一响应格式 =====
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@ApiResult(description = "API统一返回格式")
public class ApiResult<T> {
    /** 业务错误码（0=成功，非0=失败） */
    private int code;

    /** 提示信息 */
    private String msg;

    /** 数据载荷 */
    private T data;

    /** 时间戳 */
    private Long timestamp;

    /** 成功快捷方法 */
    public static <T> ApiResult<T> ok(T data) {
        return ApiResult.<T>builder()
                .code(0).msg("success").data(data)
                .timestamp(System.currentTimeMillis()).build();
    }

    /** 失败快捷方法 */
    public static <T> ApiResult<T> fail(int code, String msg) {
        return ApiResult.<T>builder()
                .code(code).msg(msg).timestamp(System.currentTimeMillis()).build();
    }
}
```

### 关键设计要点

#### 要点1：日志级别区分

```java
// 业务异常（预期内的）→ WARN 级别（不是ERROR！因为它是正常的业务分支）
@ExceptionHandler(BizException.class)
public Result handleBiz(BizException e) {
    log.warn("[业务异常] {}", e.getMessage());  // WARN
}

// 系统异常（非预期的）→ ERROR 级别（需要立即关注）
@ExceptionHandler(Exception.class)
public Result handleSys(Exception e) {
    log.error("[系统异常]", e);  // ERROR + 打印完整堆栈
}
```

#### 要点2：生产环境脱敏

```java
private boolean isProd() {
    // 通过Spring Profile判断
    return Arrays.asList(env.getActiveProfiles()).contains("prod");
}

// 生产环境返回的信息要模糊：
// prod:  {"code": 500, "msg": "系统繁忙"}
// dev:   {"code": 500, "msg": "NullPointerException: cannot invoke method on null"}
```

#### 要点3：异常处理的优先级

```java
// @ExceptionHandler 的匹配规则：
// 1. 最精确匹配优先（子类 > 父类）
// 2. 同级按方法定义顺序
//
// 所以要把最具体的异常放在前面，Exception 兜底放最后面
//
// 匹配顺序：
// ① BizException (自定义业务异常)
// ② MethodArgumentNotValidException (Spring校验异常)
// ③ ConstraintViolationException (JSR303校验异常)
// ④ ResourceNotFoundException (404)
// ⑤ AccessDeniedException (403)
// ⑥ ThirdPartyException (第三方)
// ⑦ Exception (兜底)
```

---

## Q6：自定义异常中 Throwable 的 cause 和原始异常的区别？

**A：**

### 本质上没有区别——cause 就是原始异常

```java
// Throwable 类内部维护了一个 cause 字段：
public class Throwable {
    private volatile Throwable cause;  // 这就是"原始异常"
    private String detailMessage;

    // 设置 cause 的方式有三种：
    public synchronized Throwable initCause(Throwable cause) { ... }  // 方式1：显式设置
    protected Throwable(String message, Throwable cause) { ... }      // 方式2：构造时传入
    protected Throwable(String message) { this.cause = this; }       // 方式3：默认自身
}
```

### 三种设置 cause 的方式对比

```java
public class CauseDemo {

    // ===== 方式一：构造器传入 cause（最常用！）=====
    public void methodOne() {
        try {
            riskyOperation();
        } catch (SQLException e) {
            // 把 SQLException 作为 cause 包装到自定义异常中
            throw new BizException("DB_ERR_001", "数据库查询失败", e);
            // ↑ e 就是 cause
        }
    }

    // ===== 方式二：initCause() 显式设置 =====
    public void methodTwo() {
        BizException bizEx = new BizException("DB_ERR_001", "数据库查询失败");
        try {
            riskyOperation();
        } catch (SQLException e) {
            bizEx.initCause(e);  // 手动关联 cause
            throw bizEx;
        }
    }
    // 注意：initCause 只能调用一次！再次调用会抛 IllegalStateException

    // ===== 方式三：不设 cause（丢失根因！）=====
    public void methodThree() {
        try {
            riskyOperation();
        } catch (SQLException e) {
            // ❌ 丢失了原始异常信息！
            throw new BizException("DB_ERR_001", "数据库查询失败");
            // 无法知道具体是什么SQL错误、哪个表出了问题
        }
    }

    private void riskyOperation() throws SQLException {
        // 可能抛出各种SQL异常
    }
}
```

### 如何获取完整的异常链？

```java
// 获取直接 cause
Throwable directCause = exception.getCause();

// Spring 提供的工具：获取根因（异常链最底层的原始异常）
import org.springframework.util.ExceptionUtils;

// 获取根因（递归遍历直到 getCause() 返回 null）
Throwable rootCause = ExceptionUtils.getRootCause(exception);

// 获取整个异常链列表
List<Throwable> chain = ExceptionUtils.getThrowableList(exception);
// chain: [BizException, SQLException, NetworkException]


// 手动遍历异常链
public static void printExceptionChain(Throwable t) {
    int depth = 0;
    Throwable current = t;
    while (current != null) {
        System.out.printf("%s[%d] %s: %s%n",
                "  ".repeat(depth),
                depth,
                current.getClass().getSimpleName(),
                current.getMessage());

        // 打印堆栈摘要（仅显示本层的栈帧）
        StackTraceElement[] stackTrace = current.getStackTrace();
        for (int i = 0; i < Math.min(3, stackTrace.length); i++) {
            System.out.printf("%s  at %s%n", "  ".repeat(depth), stackTrace[i]);
        }

        current = current.getCause();  // 沿着链往下走
        depth++;
    }
}
```

### cause 循环检测

```java
// JDK 内置了 cause cycle 检测机制：
// 如果你尝试将异常自身的引用设为 cause（形成环），会被拦截

Throwable t = new Exception("test");
t.initCause(t);  // 抛出 IllegalStateException: Self-causation not permitted

// 更复杂的环也能检测：
Exception a = new Exception("A");
Exception b = new Exception("B");
a.initCause(b);
b.initCause(a);  // 抛出 IllegalStateException: Caused cycle detected
```

---

## Q7：自定义异常的序列化问题（序列化异常时的注意事项）

**A：**

### 核心问题：自定义异常能正常反序列化吗？

如果你的自定义异常实现了 `Serializable`（虽然大多数情况下不会刻意去实现），就需要注意以下问题。

### 场景：异常跨 JVM 传输

```java
// 微服务场景下，异常可能通过网络从 A 服务传到 B 服务
// （例如 RPC 框架将异常对象序列化后传给调用方）

// 自定义异常
public class PaymentException extends RuntimeException implements Serializable {
    private static final long serialVersionUID = 1L;

    private String paymentId;
    private BigDecimal amount;

    // 如果这个异常被序列化并通过网络传输...

    // ⚠️ 问题1：接收方的JVM可能没有这个异常类的定义
    // 结果：ClassNotFoundException → 反序列化失败

    // ⚠️ 问题2：接收方的类版本不同（serialVersionUID 不同）
    // 结果：InvalidClassException

    // ⚠️ 问题3：异常中包含不可序列化的字段
    // 例如：private Connection dbConn;  → NotSerializableException
}

// 最佳实践：RPC/微服务场景下不要传输异常对象本身
// 而是传输异常的结构化信息（错误码+错误消息+上下文）
```

### 推荐方案：DTO 化异常信息

```java
/**
 * 异常信息 DTO —— 用于跨进程传输
 * 只包含可序列化的基本信息，不含堆栈跟踪
 */
@Data
@AllArgsConstructor
@NoArgsConstructor
public class ErrorInfo implements Serializable {
    private static final long serialVersionUID = 1L;

    private String errorCode;      // 错误码
    private String errorMessage;    // 错误消息
    private int httpStatusCode;     // HTTP状态码
    private Map<String, Object> debugContext;  // 调试上下文
    private long timestamp;         // 时间戳
}

// 全局异常处理器中将异常转为 ErrorInfo：
@ExceptionHandler(BizException.class)
public ResponseEntity<ErrorInfo> handleBiz(BizException e) {
    ErrorInfo info = new ErrorInfo(
            e.getErrorCode(),
            e.getMessage(),
            e.getHttpStatus(),
            e.getContext(),
            System.currentTimeMillis()
    );
    return ResponseEntity.status(e.getHttpStatus()).body(info);
    // ErrorInfo 是纯数据对象，可以被任何语言的客户端解析
}
```

### 如果确实需要序列化异常对象

```java
/**
 * 可安全序列化的异常基类
 */
public abstract class SerializableBizException extends RuntimeException implements Serializable {
    private static final long serialVersionUID = 1L;

    /** 只保留可序列化的基本字段 */
    private final String errorCode;
    private final String errorMessage;

    // ⚠️ 不要持有以下类型的字段引用：
    // - Connection, Statement, ResultSet (数据库资源)
    // - InputStream, OutputStream (流)
    // - Thread, ExecutorService (线程池)
    // - Logger (日志框架对象)
    // - HttpServletRequest/Response (Servlet容器对象)
    // 这些都是不可序列化的！

    protected SerializableBizException(String errorCode, String errorMessage) {
        super(errorMessage);
        this.errorCode = errorCode;
        this.errorMessage = errorMessage;
    }

    // ⚠️ 重写 writeObject/readObject 来控制序列化行为
    @Serial
    private void writeObject(java.io.ObjectOutputStream out) throws IOException {
        out.defaultWriteObject(out);  // 序列化自己的字段
        // 注意：父类 RuntimeException 的字段也会自动序列化（包括 detailMessage, cause 等）
        // 但如果 cause 不可序列化，会抛异常！
    }

    @Serial
    private void readObject(java.io.ObjectInputStream in)
            throws IOException, ClassNotFoundException {
        in.defaultReadObject(in);
        // 可以在这里做额外的初始化或验证
    }
}

// 安全的使用示例
public class SafePaymentException extends SerializableBizException {
    private static final long serialVersionUID = 2L;

    // 只包含基本类型和String
    private final String paymentId;
    private final double amount;
    private transient String sensitiveInfo;  // 敏感信息标记transient不参与序列化

    public SafePaymentException(String paymentId, double amount, String msg) {
        super("PAYMENT_ERR", msg);
        this.paymentId = paymentId;
        this.amount = amount;
        this.sensitiveInfo = "secret";  // 不会被序列化
    }
}
```
