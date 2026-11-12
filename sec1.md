# 一、它到底做了什么：三个动作

外面常把 Spring Boot 说得很玄，其实拆开就三件事。

### 动作一：把依赖收敛成 starter

以前引一个 Web 功能，你要自己凑：`spring-webmvc`、`jackson-databind`、`tomcat-embed-core`、`validation-api`……版本还得互相对得上，错一个就 `NoSuchMethodError`。

现在引一个 `spring-boot-starter-web`，它把这些全带上，版本由父 POM 统一仲裁。**starter 本身是一段空代码**——打开它的 jar，里面除了 `META-INF` 什么都没有，只有 `pom.xml` 里的 `<dependencies>`。

### 动作二：把配置推导成自动配置类

jar 进来了，但"进来了"不等于"装好了"。自动配置类负责在合适的条件下把 `DataSource`、`SqlSessionFactory`、`RedisTemplate`、`RabbitTemplate` 这些 bean 注册进容器。

这些自动配置类**本来就写在 Spring Boot 的 jar 里**，一直在 classpath 上。它们不是"被调用"的，而是被**筛选**出来的：启动时把候选清单全部读出来，逐个跑条件判断，通过的才注册。

### 动作三：把运行方式从"部署"改成"启动"

以前写完打 war 包，扔进 Tomcat 的 `webapps`，重启 Tomcat。

现在 `main()` 方法里 `SpringApplication.run()`，内嵌 Tomcat 跟着 JVM 一起起来，打成 fat jar 直接 `java -jar`。这一步改动看似只是打包方式，实际上改变了整个部署模型：一个进程一个应用，JVM 参数、日志、监控全都自己管。

### 一张表看清差别

| 维度 | Spring 2.5（《Spring 揭秘》时代） | Spring Boot |
|---|---|---|
| 装配入口 | `applicationContext.xml` | `@SpringBootApplication` |
| 谁决定装什么 | 开发者手写 | 自动配置类 + 条件判断 |
| 依赖版本 | 自己对齐 | 父 POM 仲裁 |
| 容器启动 | `new ClassPathXmlApplicationContext("...")` | `SpringApplication.run()` |
| Web 容器 | 外部 Tomcat，部署 war | 内嵌 Tomcat，跑 fat jar |
| 配置来源 | XML + properties | 命令行 / 环境变量 / yml / properties，分层覆盖 |
| 扩展方式 | 换实现类、写 XML | 写 starter、写 `@Conditional` |

### @SpringBootApplication 拆开是什么

这个注解是三个注解的合体：

```java
@SpringBootApplication
// 等价于
@SpringBootConfiguration   // 本质是 @Configuration，标记这是个配置类
@EnableAutoConfiguration   // 核心：开启自动配置
@ComponentScan             // 扫描当前包及子包
```

三者里只有 `@EnableAutoConfiguration` 是新的。`@Configuration` 和 `@ComponentScan` 在 Spring 3.0 就有了，《Spring 揭秘》后半部分讲的注解驱动就是这套。

::: tip 为什么入口类要放在最外层包
`@ComponentScan` 默认扫**当前类所在包及其子包**。入口类放在 `com.example` 下，`com.example.order`、`com.example.user` 才会被扫到；放到 `com.example.web` 里，兄弟包全扫不到，表现就是"bean 找不到"。
:::
