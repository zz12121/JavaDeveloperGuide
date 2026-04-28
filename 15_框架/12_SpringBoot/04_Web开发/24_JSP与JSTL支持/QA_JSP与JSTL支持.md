# JSP与JSTL支持 - QA

## Q1：Spring Boot 推荐使用 JSP 吗？

**A**：不推荐。

**原因**：
- Spring Boot 不支持 JSP 内嵌运行
- 必须打包为 war 部署
- Thymeleaf 更适合 Spring Boot

**建议**：新项目使用 Thymeleaf，JSP 仅用于迁移遗留项目。

---

## Q2：如何启用 JSTL？

**A**：

```jsp
<%@ taglib prefix="c" uri="http://java.sun.com/jsp/jstl/core" %>
<%@ taglib prefix="fmt" uri="http://java.sun.com/jsp/jstl/fmt" %>
```

---

## Q3：JSP 如何获取 Spring Bean？

**A**：

```jsp
<%@ page import="com.example.service.UserService" %>
<%
    UserService userService = (UserService) application.getAttribute("userService");
%>
```

**不推荐**：JSP 中直接使用 Spring Bean 破坏分层架构。
