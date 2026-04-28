# Servlet API原生对象注入 - QA

## Q1：Spring 推荐直接使用 Servlet API 吗？

**A**：不推荐，尽量使用封装好的方式。

| 场景 | 推荐方式 | 不推荐 |
|------|----------|--------|
| 获取请求参数 | @RequestParam, @PathVariable | HttpServletRequest.getParameter() |
| 获取请求头 | @RequestHeader | HttpServletRequest.getHeader() |
| 文件上传 | MultipartFile | 原始 Servlet FileUpload |
| 响应 | @ResponseBody | HttpServletResponse.getWriter() |

---

## Q2：@RequestScope 和 HttpServletRequest 的区别？

**A**：

| 对比 | @RequestScope Bean | HttpServletRequest 参数 |
|------|-------------------|-------------------------|
| 注入方式 | @Autowired | 方法参数 |
| 生命周期 | 整个请求共享 | 局部参数 |
| 可测试性 | 高 | 低 |

```java
// 推荐：方法参数（推荐）
@GetMapping("/user")
public User getUser(HttpServletRequest request) {
    String id = request.getParameter("id");
    return userService.getById(Long.parseLong(id));
}

// 可选：@RequestScope Bean
@Component
@RequestScope
public class UserContext {
    private HttpServletRequest request;
    
    public String getId() {
        return request.getParameter("id");
    }
}
```

---

## Q3：如何在非 Controller 中获取 HttpServletRequest？

**A**：三种方式。

```java
// 方式1：HttpServletRequestAware
@Component
public class RequestAwareBean implements HttpServletRequestAware {
    private HttpServletRequest request;
    
    @Override
    public void setHttpServletRequest(HttpServletRequest request) {
        this.request = request;
    }
}

// 方式2：RequestContextHolder（不推荐，耦合 Servlet API）
HttpServletRequest request = 
    ((ServletRequestAttributes) RequestContextHolder.getRequestAttributes())
        .getRequest();

// 方式3：注入到方法参数（推荐）
@Service
public class UserService {
    public void process(HttpServletRequest request) {
        // 使用
    }
}
```

---

## Q4：HttpSession 和 @SessionAttributes 的区别？

**A**：

| 对比 | HttpSession | @SessionAttributes |
|------|-------------|-------------------|
| 范围 | 整个会话 | 仅当前 Controller |
| 存储 | 任意对象 | Model 属性 |
| 生命周期 | 浏览器关闭或超时 | 直到 Controller 处理完成 |

```java
// HttpSession：跨 Controller 共享
@GetMapping("/set")
public String setSession(HttpSession session) {
    session.setAttribute("user", currentUser);
    return "ok";
}

@GetMapping("/get")
public String getSession(HttpSession session) {
    User user = (User) session.getAttribute("user");
    return user.getName();
}
```

---

## Q5：如何获取客户端 IP？

**A**：

```java
@GetMapping("/ip")
public String getClientIp(HttpServletRequest request) {
    String ip = request.getHeader("X-Forwarded-For");
    if (ip == null || ip.isEmpty() || "unknown".equalsIgnoreCase(ip)) {
        ip = request.getHeader("Proxy-Client-IP");
    }
    if (ip == null || ip.isEmpty() || "unknown".equalsIgnoreCase(ip)) {
        ip = request.getHeader("WL-Proxy-Client-IP");
    }
    if (ip == null || ip.isEmpty() || "unknown".equalsIgnoreCase(ip)) {
        ip = request.getRemoteAddr();
    }
    // 多级代理时取第一个 IP
    if (ip != null && ip.contains(",")) {
        ip = ip.split(",")[0].trim();
    }
    return ip;
}
```
