---
title: "QA_反射获取注解与继承关系"
module: 08_反射
created: 2026-04-25
---

# 面试题：反射获取注解与继承关系

### Q1：getAnnotation() 和 getDeclaredAnnotation() 的核心区别是什么？

**参考答案：**

| 方法 | 是否查继承 | 适用场景 |
|------|:---:|------|
| `getAnnotation(Class)` | ✅ 查 | 想知道"最终生效"的注解（含继承的） |
| `getDeclaredAnnotation(Class)` | ❌ 不查 | 只看"自己写的"注解 |

```java
@InheritedAnno
class Parent {}

class Child extends Parent {}

// 查继承的注解
Child.class.getAnnotation(InheritedAnno.class);    // ✅ 找到
Child.class.getDeclaredAnnotation(InheritedAnno.class); // ❌ 找不到

// 查直接声明的注解（不管 @Inherited）
Parent.class.getAnnotation(InheritedAnno.class);    // ✅ 找到
Parent.class.getDeclaredAnnotation(InheritedAnno.class); // ✅ 找到
```

**记忆口诀**：`Declared` = "我只看我自己的，不管继承来的"。

---

### Q2：@Inherited 注解的继承规则是什么？有什么限制？

**参考答案：**

**@Inherited 只对类有效，对方法、字段、构造函数无效。**

```java
@Inherited
@interface InheritedAnno {}

@InheritedAnno
class Parent {
    @InheritedAnno  // 这个注解不会被子类方法继承
    public void method() {}
}

class Child extends Parent {
    // Child.method() 上的 @InheritedAnno 来自子类自己声明
    // 不是继承的，因为 @Inherited 对方法无效
}
```

**完整继承规则表：**

| 注解位置 | @Inherited 有效？ | 子类是否继承 |
|---------|:---:|:---:|
| 类上标注 | ✅ | 继承（但子类 getDeclaredAnnotation 查不到） |
| 方法上标注 | ❌ | 不继承 |
| 字段上标注 | ❌ | 不继承 |
| 接口上标注 | ❌ | 实现类不继承（除非接口本身被继承） |

---

### Q3：MyBatis 为什么用 getDeclaredAnnotation 而不是 getAnnotation？

**参考答案：**

**MyBatis 的 Mapper 接口方法上标注的 @Select/@Update/@Insert/@Delete 不会被继承**，因为：

1. **接口方法不继承注解**：Java 语法规定接口实现类不继承接口方法的注解
2. **避免父接口的注解被误用**：如果父接口也有同名方法，可能导致混淆

```java
// MyBatis 内部实现（简化）
interface Mapper {
    @Select("SELECT * FROM user WHERE id = #{id}")
    User selectById(Long id);
}

// 通过反射获取方法上的注解
Method method = Mapper.class.getDeclaredMethod("selectById", Long.class);
// 不用 getAnnotation，因为接口方法注解不会被继承
Select select = method.getDeclaredAnnotation(Select.class);
```

如果用了 `getAnnotation()`，即使子类 Mapper 声明了 `@Select("different sql")`，也可能因为继承机制产生意外结果。

---

### Q4：Spring MVC 如何用反射找到请求映射方法？

**参考答案：**

Spring MVC 的 `RequestMappingHandlerMapping` 核心逻辑：

```java
// 1. 获取当前类所有声明的方法
Method[] methods = controllerClass.getDeclaredMethods();

for (Method method : methods) {
    // 2. 只看方法自己声明的注解（不用 getAnnotation）
    RequestMapping rm = method.getDeclaredAnnotation(RequestMapping.class);
    GetMapping gm = method.getDeclaredAnnotation(GetMapping.class);
    PostMapping pm = method.getDeclaredAnnotation(PostMapping.class);

    if (rm != null || gm != null || pm != null) {
        // 3. 注册到映射表：URL → Method
        registerMapping(urlPattern, method, bean);
    }
}

// 4. 父类的方法（如果有 @RequestMapping）也需要处理
Method[] parentMethods = controllerClass.getMethods();
for (Method method : parentMethods) {
    // 父类公共方法上的注解需要单独处理
    RequestMapping rm = method.getAnnotation(RequestMapping.class);
    if (rm != null && Modifier.isPublic(method.getModifiers())) {
        registerMapping(...);
    }
}
```

**关键点**：
- `getDeclaredMethods()` 获取自身声明的所有方法（含 private/protected）
- `getDeclaredAnnotation()` 确保只获取方法自身标注的映射注解
- `getMethods()` 获取继承/实现的公共方法（单独处理）

---

### Q5：如何写一个自定义的权限校验注解框架？

**参考答案：**

```java
// 注解定义
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@interface RequirePermission {
    String value();
}

// 拦截器实现
@Aspect
@Component
public class PermissionInterceptor {

    @Around("@annotation(RequirePermission)")
    public Object checkPermission(ProceedingJoinPoint pjp) throws Throwable {
        MethodSignature signature = (MethodSignature) pjp.getSignature();
        Method method = signature.getMethod();

        // 用 getDeclaredAnnotation 获取方法级权限注解
        RequirePermission rp = method.getDeclaredAnnotation(RequirePermission.class);

        if (rp == null) {
            // 方法级没有，查类级
            Class<?> declaringClass = method.getDeclaringClass();
            rp = declaringClass.getDeclaredAnnotation(RequirePermission.class);
        }

        if (rp == null) {
            throw new SecurityException("缺少权限注解");
        }

        // 校验用户权限
        User currentUser = SecurityContext.getCurrentUser();
        if (!currentUser.hasPermission(rp.value())) {
            throw new SecurityException("权限不足: " + rp.value());
        }

        return pjp.proceed();
    }
}

// 使用
@RequirePermission("user:read")
public User getUser(Long id) { ... }
```

**面试加分点**：说明为什么 `getDeclaredAnnotation` 比 `getAnnotation` 更安全——避免父类或接口的注解被意外继承导致权限误判。
