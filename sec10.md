# 十、十个高频坑

按踩到的频率排序。

### 1. Bean 覆盖默认被禁止

Spring Boot 2.1 起，`spring.main.allow-bean-definition-overriding` 默认 `false`。两个同名 bean 会直接启动失败：

```
The bean 'xxx', defined in class path resource [...], could not be registered.
A bean with that name has already been defined and overriding is disabled.
```

这是好事——它把隐藏的覆盖问题暴露出来了。正确做法是改名或加 `@Primary`，而不是打开开关。

### 2. 循环依赖默认被禁止

Spring Boot 2.6 起，`spring.main.allow-circular-references` 默认 `false`。

```
The dependencies of some of the beans in the application context form a cycle
```

以前靠三级缓存能"糊"过去（仅限 setter/field 注入），现在直接报错。**正确做法是重构**：抽第三层、用 `@Lazy` 延迟一个、或者改成事件驱动。打开开关只是把问题推到运行期。

### 3. @Transactional / @Async 自调用失效

```java
@Service
public class OrderService {
    public void create() {
        this.doCreate();       // ← 不走代理，事务注解无效
    }

    @Transactional
    public void doCreate() { ... }
}
```

原因是 `this` 不是代理对象。三种解法：

- 把 `doCreate` 挪到另一个 bean
- `AopContext.currentProxy()`（需开 `exposeProxy`）
- 注入自己（`@Lazy @Autowired OrderService self`）

### 4. 配置文件编码

前面讲过 GBK 的坑。补充一点：`application.yml` 同样有这个问题，而且 YAML 对缩进敏感，编码错了往往报的是"缩进非法"，更难定位。

### 5. @ConfigurationProperties 不生效

三个常见原因：

- 类没被注册成 bean（没 `@Component`，也没 `@EnableConfigurationProperties`）
- prefix 写错（`@ConfigurationProperties(prefix="redis")` 但配置写的是 `spring.redis.host`）
- 字段没有 setter（用了 Lombok 但没加 `@Data`）

### 6. 静态资源 404

Spring Boot 默认从四个位置找静态资源：

```
classpath:/static/
classpath:/public/
classpath:/resources/
classpath:/META-INF/resources/
```

放在 `src/main/resources/static/` 下才生效。放在 `webapp/` 下（Maven 传统目录）不生效，除非打 war。

### 7. 端口占用但报错看不懂

```
Web server failed to start. Port 8080 was already in use.
```

这个还算友好。更坑的是报 `java.net.BindException: Address already in use` 却不说哪个端口——通常是 `management.server.port` 或某个中间件客户端端口冲突。

排查：`lsof -i :8080`。

### 8. 日志配置不生效

`logback.xml` 和 `logback-spring.xml` 的区别：后者支持 `<springProfile>` 和 `<springProperty>`，前者不支持。用了 Spring Boot 就用 `-spring` 版本。

另外，`logging.config` 指定的文件路径在 fat jar 里要用 `classpath:` 前缀。

### 9. devtools 的类加载器陷阱

`spring-boot-devtools` 用两个类加载器：`base`（加载不变的依赖）和 `restart`（加载你的 class）。这导致：

- 静态字段在重启后**会重置**（因为类被重新加载了）
- `ThreadLocal` 里的东西可能残留
- 某些缓存（比如 `ConcurrentHashMap` 静态实例）行为异常

生产环境务必排除 devtools：

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-devtools</artifactId>
  <optional>true</optional>
  <scope>runtime</scope>
</dependency>
```

`<optional>true</optional>` 保证它不会传递到最终产物。

### 10. 打包后找不到 resources 下的文件

开发时 `new FileInputStream("src/main/resources/x.txt")` 能跑，打包后必挂。因为 jar 里的资源不是文件。

统一用：

```java
// 方式一：流
try (InputStream in = new ClassPathResource("x.txt").getInputStream()) { ... }

// 方式二：需要遍历时
Resource[] rs = new PathMatchingResourcePatternResolver()
        .getResources("classpath*:templates/*.html");
```

## 结语：Spring Boot 之后该看什么

这本书讲完了三个动作：starter 收敛依赖、自动配置推导装配、内嵌容器改变运行方式。加上条件机制和启动流程，Spring Boot 的"为什么"就齐了。

再往下走，有三个方向：

**方向一：读 Spring Boot 源码。**
重点是 `spring-boot-project/spring-boot-autoconfigure` 这个模块。它没有复杂算法，全是"条件 + 注册"，读起来比 Spring Framework 轻松得多。建议从 `DataSourceAutoConfiguration`、`RedisAutoConfiguration`、`RabbitAutoConfiguration` 这三个入手——它们的结构最典型，而且跟你手上的秒杀项目直接对应。

**方向二：读中间件客户端源码。**
自动配置只是把客户端"装"进容器，真正的坑都在客户端本身：Jedis 的连接池怎么借还、RabbitMQ 的 channel 怎么复用、MyBatis 的 `SqlSession` 什么时候关。这部分我在《中间件实战：Redis + RabbitMQ》里展开。

**方向三：读 Spring Cloud。**
微服务那一套（注册中心、配置中心、网关、熔断）本质上是"在 Spring Boot 之上再套一层自动配置"。看懂这本，Spring Cloud 的 `@EnableDiscoveryClient`、`@EnableFeignClients` 就是同一个套路。

::: tip 最后一句
Spring Boot 让人觉得"神奇"，是因为看不到显式的配置。但神奇感消失得很快——你只要记住一句话：**所有"自动"的背后，都有一份清单、一堆条件、一次排序。** 找不到 bean 的时候，去 `--debug` 里看条件报告，答案总在那里。
:::
