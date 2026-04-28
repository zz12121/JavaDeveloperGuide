# 懒加载初始化

## Q1：懒加载和立即加载有什么区别？

**A**：

| 对比项 | 立即加载（默认） | 懒加载 |
|--------|-----------------|--------|
| **创建时机** | 启动时创建所有 Bean | 首次使用时创建 |
| **启动速度** | 慢 | 快 |
| **内存占用** | 启动时全部加载 | 按需加载 |
| **错误暴露** | 启动时暴露 | 运行时暴露 |
| **首次响应** | 快 | 慢 |

**立即加载流程**：
```
启动 → 创建 DataSource → 创建 SqlSessionFactory → 创建 Service → 启动完成
```

**懒加载流程**：
```
启动 → 启动完成 → 首次请求 → 创建 DataSource → ... → 返回响应
```

---

## Q2：如何启用懒加载？

**A**：有两种方式：

**方式一：配置文件**
```yaml
spring:
  main:
    lazy-initialization: true
```

**方式二：代码**
```java
new SpringApplication(App.class)
    .setLazyInitialization(true);
```

---

## Q3：@Lazy 注解如何使用？

**A**：

```java
// 1. 在 Bean 类上
@Lazy
@Component
public class MyService { }

// 2. 在 @Bean 方法上
@Bean
@Lazy
public MyBean myBean() {
    return new MyBean();
}

// 3. 注入时
@Bean
public MyConfig {
    @Lazy
    @Autowired
    private MyService myService;
}

// 4. 构造器注入
@Service
public class UserService {
    public UserService(@Lazy MyService myService) {
        this.myService = myService;
    }
}
```

---

## Q4：懒加载会影响哪些 Bean？

**A**：

**不受影响的**：
- `@Configuration` 类本身（非 @Bean 方法）
- `EnvironmentPostProcessor`
- `BeanFactoryPostProcessor`
- `ApplicationContextInitializer`

**受影响的**：
- 所有 `@Component`、`@Service`、`@Repository`、`@Controller`
- 所有 `@Bean` 方法
- `@Autowired` 注入的 Bean

---

## Q5：懒加载有什么风险？

**A**：

| 风险 | 说明 |
|------|------|
| **错误延迟暴露** | 配置错误直到首次使用时才抛出 |
| **循环依赖难检测** | 懒加载可能掩盖循环依赖问题 |
| **事务问题** | 懒加载的 Bean 在事务上下文中可能有问题 |
| **首次请求慢** | 用户首次请求时需要等待 Bean 创建 |

**示例 - 配置错误延迟暴露**：
```yaml
# 错误的数据库配置
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/xxx  # 这个数据库不存在
```

**立即加载**：启动失败，立刻知道配置错误
**懒加载**：启动成功，直到第一次查询数据库才失败

---

## Q6：如何针对单个 Bean 禁用懒加载？

**A**：

```java
@Configuration
public class AppConfig {
    
    // 全局懒加载，但这个 Bean 立即加载
    @Bean
    @Lazy(false)
    public DataSource dataSource() {
        return DataSourceBuilder.create().build();
    }
}
```

---

## Q7：懒加载适合什么场景？

**A**：

| 场景 | 适用性 |
|------|--------|
| **开发环境** | ✅ 适合，加快启动速度 |
| **测试环境** | ✅ 适合，按需加载 |
| **生产环境** | ⚠️ 谨慎，可能影响首次请求 |
| **微服务** | ✅ 适合，加快容器启动 |
| **大项目** | ✅ 适合，减少启动内存 |

**推荐做法**：
```yaml
# 开发环境启用
spring:
  main:
    lazy-initialization: ${LAZY_INIT:false}
```

运行时通过环境变量控制。
