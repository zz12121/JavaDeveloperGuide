# Spring Boot 配置管理知识地图

> 涵盖 Spring Boot 配置管理核心知识点，共 **29个**。

---

## 一、配置文件基础

| # | 知识点 |
|---|--------|
| 1 | **application.properties 格式** — 键值对、注释、编码 |
| 2 | **application.yml / YAML 格式** — 层级结构、缩进规范 |
| 3 | **YAML 多文档配置** — `---` 分隔多文档、文档合并规则 |
| 4 | **properties vs YAML 对比** — 格式差异、优缺点、选择建议 |
| 5 | **@Value 注解读取配置** — 语法、默认值、支持类型 |
| 6 | **@Value ${} vs #{} 区别** — 属性占位符 vs SpEL 表达式 |
| 7 | **@ConfigurationProperties 配置绑定** — 批量绑定、类型安全 |
| 8 | **@EnableConfigurationProperties** — 启用配置属性类 |
| 9 | **@ConfigurationProperties vs @Value** — 功能对比、适用场景 |
| 10 | **@NestedConfigurationProperty** — 嵌套配置属性标记 |

---

## 二、外部配置加载

| # | 知识点 |
|---|--------|
| 11 | **@PropertySource 加载外部配置** — 加载指定 properties 文件 |
| 12 | **@PropertySources 多文件加载** — 加载多个外部配置文件 |
| 13 | **@ImportResource 导入 XML** — 导入 Spring XML 配置 |

---

## 三、配置优先级与环境

| # | 知识点 |
|---|--------|
| 14 | **Spring Boot 配置优先级** — 15级优先级、覆盖规则 |
| 15 | **外部化配置（Externalized Configuration）** — 配置外部化理念 |
| 16 | **命令行参数配置** — `--key=value` 方式传入参数 |
| 17 | **环境变量自动映射** — 大写下划线自动转小写横线 |
| 18 | **RandomValuePropertySource 随机值** — `${random.int}` 等语法 |
| 19 | **多环境配置** — dev/test/prod 环境隔离理念 |
| 20 | **@Profile 激活环境** — 类/方法级别环境激活 |
| 21 | **spring.profiles.active 激活** — 配置文件/命令行激活 |
| 22 | **application-{profile}.properties/yml** — profile 配置文件 |
| 23 | **spring.config.activate.on-profile（2.4+）** — 文档级激活条件 |
| 24 | **spring.config.import 导入配置** — 导入额外配置文件 |

---

## 四、高级配置特性

| # | 知识点 |
|---|--------|
| 25 | **配置占位符** — `${app.name:默认值}` 语法 |
| 26 | **Relaxed 属性名绑定** — 驼峰、横线、下划线自动兼容 |
| 27 | **配置元数据** — spring-configuration-metadata.json 自动生成 |
| 28 | **配置加密（Jasypt）** — 敏感配置加密处理 |
| 29 | **Spring Boot 2.4 配置变更总结** — 2.4 版本配置相关变化 |

---

## 📁 对应文件夹结构

```
03_配置管理/
├── 00_SpringBoot配置管理知识地图.md
├── 01_application_properties格式/
├── 02_application_yaml格式/
├── 03_YAML多文档配置/
├── 04_properties_vs_YAML对比/
├── 05_@Value注解读取配置/
├── 06_@Value_${}_vs_#{}区别/
├── 07_@ConfigurationProperties配置绑定/
├── 08_@EnableConfigurationProperties/
├── 09_@ConfigurationProperties_vs_@Value/
├── 10_@NestedConfigurationProperty/
├── 11_@PropertySource加载外部配置/
├── 12_@PropertySources多文件加载/
├── 13_@ImportResource导入XML/
├── 14_SpringBoot配置优先级/
├── 15_外部化配置/
├── 16_命令行参数配置/
├── 17_环境变量自动映射/
├── 18_RandomValuePropertySource随机值/
├── 19_多环境配置/
├── 20_@Profile激活环境/
├── 21_spring_profiles_active激活/
├── 22_application_profile配置文件/
├── 23_spring_config_activate_on_profile/
├── 24_spring_config_import导入配置/
├── 25_配置占位符/
├── 26_Relaxed属性名绑定/
├── 27_配置元数据/
├── 28_配置加密/
└── 29_SpringBoot2_4配置变更/

合计：29 个知识点
```

---

*最后更新：2026-04-28*
