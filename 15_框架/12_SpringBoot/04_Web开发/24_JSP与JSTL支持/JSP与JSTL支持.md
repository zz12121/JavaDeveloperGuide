# JSP与JSTL支持

## 先说结论

Spring Boot 支持 JSP，但需要额外配置。**JSTL 提供 JSP 的标准标签库**。

## 深度解析

### 依赖配置

```xml
<dependency>
    <groupId>org.apache.tomcat.embed</groupId>
    <artifactId>tomcat-embed-jasper</artifactId>
</dependency>
<dependency>
    <groupId>javax.servlet</groupId>
    <artifactId>jstl</artifactId>
</dependency>
```

### JSTL 核心标签

```jsp
<%@ taglib prefix="c" uri="http://java.sun.com/jsp/jstl/core" %>

<!-- 条件判断 -->
<c:if test="${user.active}">
    <p>激活</p>
</c:if>

<!-- 循环 -->
<c:forEach var="user" items="${users}">
    <p>${user.name}</p>
</c:forEach>

<!-- 格式化和函数 -->
<%@ taglib prefix="fmt" uri="http://java.sun.com/jsp/jstl/fmt" %>
<fmt:formatDate value="${user.createTime}" pattern="yyyy-MM-dd"/>
```

## 易错点/踩坑

- ❌ Spring Boot 打 war 包部署才能使用 JSP
- ❌ JSP 无法放在 jar 包中，必须 war 部署
- ❌ 缺少 tomcat-embed-jasper 依赖 → JSP 无法解析

## 关联知识点

- `23_视图解析器InternalResourceViewResolver`：视图解析配置
