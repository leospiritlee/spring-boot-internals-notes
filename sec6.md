# 六、配置系统：优先级、松散绑定、Profile

### 配置源的优先级

从高到低（后面的覆盖前面的）：

| 优先级 | 来源 | 例子 |
|---:|---|---|
| 1 | 命令行参数 | `java -jar app.jar --server.port=9090` |
| 2 | `SPRING_APPLICATION_JSON` | 环境变量里塞一段 JSON |
| 3 | Servlet 参数 / `ServletContext` 属性 | 只在 war 部署时有 |
| 4 | JNDI | `java:comp/env` |
| 5 | 系统属性 | `-Dserver.port=9090` |
| 6 | 操作系统环境变量 | `SERVER_PORT=9090` |
| 7 | `application-{profile}.properties/yml`（jar 外） | 与 jar 同目录的 config 文件夹 |
| 8 | `application-{profile}.properties/yml`（jar 内） | `src/main/resources` |
| 9 | `application.properties/yml`（jar 外） | |
| 10 | `application.properties/yml`（jar 内） | |
| 11 | `@PropertySource` 注解 | |
| 12 | 默认值 | `SpringApplication.setDefaultProperties()` |

::: warn 最容易被坑的一条
**环境变量会覆盖配置文件。** 你在 `application.properties` 里写 `server.port=8080`，但如果机器上有个环境变量 `SERVER_PORT=9090`（比如别的项目留下的），端口就是 9090。

排查方法：`--debug` 启动，或者访问 `/actuator/env` 看 `server.port` 的实际来源。
:::

### 配置文件加载位置

`ConfigDataEnvironmentPostProcessor` 会按顺序找这四个位置：

```
1. classpath:/                           （jar 内 resources 根）
2. classpath:/config/                    （jar 内 resources/config）
3. file:./                                （jar 所在目录）
4. file:./config/                         （jar 所在目录的 config 子目录）
4. file:./config/*/                       （config 下的直接子目录，2.4+）
```

**后面的覆盖前面的。** 所以运维改配置的标准做法是：在 jar 同目录建 `config/application.properties`，不动 jar 本身。

### 松散绑定

同一个配置项，可以有多种写法：

```properties
# 以下四种写法都能绑到同一个字段
my.app.user-name=leo
my.app.userName=leo
my.app.user_name=leo
my.app.USER_NAME=leo
```

对应的环境变量形式是 `MY_APP_USERNAME`（点换下划线，横线去掉，全大写）。

这带来一个隐蔽问题：**`@Value("${my.app.user-name}")` 不支持松散绑定**，只认完全一致的 key。而 `@ConfigurationProperties` 支持。这是选型理由之一。

### @ConfigurationProperties vs @Value

| 维度 | `@ConfigurationProperties` | `@Value` |
|---|---|---|
| 批量绑定 | 支持，一个类绑一组 | 不支持，一个字段一个 |
| 松散绑定 | 支持 | 不支持 |
| 类型转换 | 自动，支持嵌套对象、List、Map | 只支持简单类型 |
| 校验 | 支持 `@Validated` + JSR-303 | 不支持 |
| SpEL | 不支持 | 支持 `#{...}` |
| IDE 提示 | 生成 `spring-configuration-metadata.json` 后能补全 | 无 |
| 适用 | 一组相关配置 | 单个值、需要表达式 |

秒杀项目里的写法就很标准：

```java
@Data
@Component
@ConfigurationProperties(prefix = "redis")
public class RedisConfig {
    private String host;
    private int port;
    private int timeout;
    private String password;
    private int poolMaxTotal;
    private int poolMaxIdle;
    private int poolMaxWait;
}
```

对应 `application.properties`：

```properties
redis.host=127.0.0.1
redis.port=6379
redis.timeout=3
redis.poolMaxTotal=1000
redis.poolMaxIdle=200
redis.poolMaxWait=3
```

::: tip @Component 不是必须的
不加 `@Component`，改用 `@EnableConfigurationProperties(RedisConfig.class)` 或 `@ConfigurationPropertiesScan` 也能注册。好处是配置类可以保持"纯 POJO"，不依赖 Spring 注解——这在写 starter 时很重要，因为 starter 的使用者不一定想让你扫他的包。
:::

### 类型安全与校验

```java
@Validated
@ConfigurationProperties(prefix = "app.mail")
public class MailProperties {
    @NotBlank
    private String host;
    @Min(1) @Max(65535)
    private int port = 25;
    @Email
    private String from;
    private List<String> cc = new ArrayList<>();
}
```

启动时校验不过直接启动失败，比运行到一半才 NPE 好得多。

### Profile

```java
@Configuration
@Profile("dev")
public class DevConfig { }

@Configuration
@Profile({"prod", "staging"})
public class ProdConfig { }

@Configuration
@Profile("!test")     // 非 test 环境
public class NotTestConfig { }
```

激活方式：

```properties
spring.profiles.active=dev
```

或者启动参数 `--spring.profiles.active=dev`，或者环境变量 `SPRING_PROFILES_ACTIVE=dev`。

2.4 之后还支持 profile group：

```properties
spring.profiles.group.prod=prod-db,prod-redis,prod-mq
```

激活 `prod` 等于同时激活三个子 profile。

::: warn 一个真实的编码坑
秒杀项目的 `application.properties` 里有一行中文注释，文件存成了 **GBK**。Spring Boot 2.x 默认按 **UTF-8** 读配置文件，结果那一行乱码，后面的 `redis.password=` 被吞掉，启动时 `JedisPool` 拿到 null 密码连不上 Redis。

排查方法：`file -I application.properties` 看编码，或者用 `xxd` 看有没有非 UTF-8 字节序列。

修复：统一转成 UTF-8，并在 `pom.xml` 里设 `<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>`。
:::
