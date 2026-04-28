# @SessionAttributes和@SessionAttribute

## 先说结论

`@SessionAttributes` 将模型数据存储到会话，`@SessionAttribute` 从会话中获取数据。**实现跨请求数据共享**。

## 深度解析

### @SessionAttributes

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
public @interface SessionAttributes {
    @AliasFor("value")
    String[] names() default {};
    
    @AliasFor("value")
    String[] value() default {};
    
    Class<?>[] types() default {};
}
```

### @SessionAttribute

```java
@Target({ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface SessionAttribute {
    @AliasFor("value")
    String name() default "";
    
    @AliasFor("name")
    String value() default "";
    
    boolean required() default true;
}
```

### 基本使用

```java
// 类级别：声明要存入会话的属性
@SessionAttributes({"user", "token"})
@Controller
public class UserController {
    
    // 存储到会话
    @PostMapping("/login")
    public String login(@RequestParam String username, Model model) {
        User user = userService.login(username);
        model.addAttribute("user", user);  // 存入会话
        return "redirect:/dashboard";
    }
    
    // 从会话获取
    @GetMapping("/dashboard")
    public String dashboard(@SessionAttribute("user") User user) {
        return "dashboard";
    }
}
```

### 存储多种类型

```java
@SessionAttributes(types = {User.class, String.class})
@Controller
public class UserController {
    // 所有 User 类型和 String 类型都会存入会话
}
```

## 易错点/踩坑

- ❌ @SessionAttributes 用于 @RestController → 不生效，@RestController 无视图
- ❌ 存储过多会话数据 → 占用内存，建议只存必要数据
- ❌ 会话超时 → 数据丢失，需处理 null

## 代码示例

### 购物车场景

```java
@SessionAttributes({"cart"})
@Controller
public class CartController {
    
    @ModelAttribute("cart")
    public Cart createCart() {
        return new Cart();
    }
    
    @PostMapping("/add")
    public String addToCart(
        @ModelAttribute Cart cart,
        @RequestParam Long productId
    ) {
        cart.addItem(productService.getById(productId));
        return "redirect:/cart/view";
    }
    
    @GetMapping("/view")
    public String viewCart(@SessionAttribute Cart cart, Model model) {
        model.addAttribute("cart", cart);
        return "cart/view";
    }
}
```

## 关联知识点

- `17_处理模型数据`：Model 数据处理
- `19_RedirectAttributes重定向数据`：重定向时的数据传递
