# SpringBoot Web开发知识地图

## 模块简介

本模块涵盖 SpringBoot Web 开发的核心知识点，包括自动配置、请求处理、视图技术、静态资源、文件上传下载、异常处理等。

## 知识点索引

| # | 知识点 | 优先级 |
|---|--------|--------|
| 01 | SpringBoot自动配置WebMvc | 🔴 高 |
| 02 | @SpringBootApplication注解三合一 | 🔴 高 |
| 03 | WebMvcAutoConfiguration自动配置原理 | 🔴 高 |
| 04 | DispatcherServlet自动注册 | 🟡 中 |
| 05 | HttpMessageConverters自动配置 | 🟡 中 |
| 06 | ContentNegotiation内容协商 | 🟡 中 |
| 07 | @RequestMapping请求映射 | 🔴 高 |
| 08 | @GetMapping @PostMapping等快捷注解 | 🔴 高 |
| 09 | @PathVariable路径变量 | 🔴 高 |
| 10 | @RequestParam请求参数 | 🔴 高 |
| 11 | @RequestBody请求体 | 🔴 高 |
| 12 | @ResponseBody响应体 | 🔴 高 |
| 13 | @RestController复合注解 | 🔴 高 |
| 14 | @RequestHeader @CookieValue | 🟡 中 |
| 15 | @ModelAttribute接收表单 | 🟡 中 |
| 16 | Servlet API原生对象注入 | 🟡 中 |
| 17 | 处理模型数据（Model/ModelMap/ModelAndView） | 🔴 高 |
| 18 | @SessionAttributes @SessionAttribute | 🟡 中 |
| 19 | RedirectAttributes重定向数据 | 🟡 中 |
| 20 | Thymeleaf模板引擎 | 🔴 高 |
| 21 | Thymeleaf标准表达式语法 | 🔴 高 |
| 22 | Thymeleaf条件渲染/循环 | 🔴 高 |
| 23 | 视图解析器InternalResourceViewResolver | 🔴 高 |
| 24 | JSP与JSTL支持 | 🟢 低 |
| 25 | 静态资源访问规则 | 🔴 高 |
| 26 | 静态资源映射路径配置 | 🟡 中 |
| 27 | WelcomePage欢迎页 | 🟡 中 |
| 28 | Favicon图标配置 | 🟢 低 |
| 29 | 文件上传配置与原理 | 🔴 高 |
| 30 | MultipartFile文件上传 | 🔴 高 |
| 31 | 文件下载实现 | 🟡 中 |
| 32 | 全局异常处理器 | 🔴 高 |
| 33 | @ExceptionHandler局部异常处理 | 🟡 中 |
| 34 | @ControllerAdvice全局异常处理 | 🔴 高 |
| 35 | @RestControllerAdvice复合注解 | 🔴 高 |
| 36 | 自定义ErrorAttributes | 🟡 中 |

---

## 复习建议

- **必背**：`@RequestMapping`、`@GetMapping`、`@PathVariable`、`@RequestParam`、`@RequestBody`、`@ResponseBody`、`@RestController`、`Thymeleaf`、`静态资源`、`文件上传`、`@ControllerAdvice`
- **理解**：自动配置原理、内容协商、视图解析、异常处理流程
- **了解**：JSP/JSTL、Favicon、文件下载细节

## 关联模块

- `02_自动配置原理`：WebMvcAutoConfiguration 原理
- `03_配置管理`：文件上传大小配置
- `11_SpringBoot3.x新特性`：WebMvc新特性
