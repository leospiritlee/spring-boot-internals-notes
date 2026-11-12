# 四、条件装配：@Conditional 家族

自动配置类能被"筛"，靠的是条件注解。这一章把条件机制讲透。

### 最底层的接口

```java
@FunctionalInterface
public interface Condition {
    boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata);
}
```

就一个方法。`ConditionContext` 给你四样东西：

| 方法 | 能拿到什么 | 典型用途 |
|---|---|---|
| `getEnvironment()` | 所有配置属性 | 读 `spring.datasource.url` 判断配没配 |
| `getBeanFactory()` | 当前 BeanFactory | 查容器里有没有某个 bean |
| `getClassLoader()` | 类加载器 | 查 classpath 有没有某个类 |
| `getRegistry()` | BeanDefinitionRegistry | 查 bean 定义（比 `getBeanFactory` 更早可用） |

返回值 `true` 表示"条件成立，这个配置类/bean 要装"。

### 常用条件注解一览

| 注解 | 判断什么 | 典型场景 |
|---|---|---|
| `@ConditionalOnClass` | classpath 有指定类 | 有 `DataSource` 才装数据源配置 |
| `@ConditionalOnMissingClass` | classpath 没有指定类 | 没引 WebFlux 才装 MVC |
| `@ConditionalOnBean` | 容器里有指定 bean | 有 `DataSource` 才装 `SqlSessionFactory` |
| `@ConditionalOnMissingBean` | 容器里没有指定 bean | 给用户留覆盖余地 |
| `@ConditionalOnProperty` | 某个配置项等于/不等于某值 | `spring.redis.enabled=true` |
| `@ConditionalOnWebApplication` | 是 Web 应用 | 只在 Servlet 环境装过滤器 |
| `@ConditionalOnNotWebApplication` | 不是 Web 应用 | 命令行工具专用配置 |
| `@ConditionalOnResource` | 某个资源文件存在 | 有 `logback-spring.xml` 才定制日志 |
| `@ConditionalOnExpression` | SpEL 表达式为真 | 复杂组合判断 |
| `@ConditionalOnJava` | JDK 版本范围 | 只在 JDK 17+ 启用的优化 |
| `@ConditionalOnSingleCandidate` | 容器里恰好一个（或有 `@Primary`） | 自动装配唯一候选 |

### 三个容易踩的细节

**细节一：`@ConditionalOnClass` 不加载类。**

它用的是 ASM 直接读字节码里的注解元数据，`Class.forName` 一次都不调。原因很实际：如果自动配置类自己 `@ConditionalOnClass(DataSource.class)`，而这个类不存在，加载配置类本身就会 `NoClassDefFoundError`，还没轮到条件判断就崩了。

所以有个写法上的讲究：**条件注解里的类，要用 `name` 字符串形式**，或者把整个配置类标在 `@ConditionalOnClass` 上而不是方法上。

```java
// 推荐：写在类上，Spring 会用 ASM 读
@ConditionalOnClass(name = "com.alibaba.druid.pool.DruidDataSource")
public class DruidConfig { }

// 危险：方法参数类型直接引用不存在的类
@Bean
@ConditionalOnClass(DruidDataSource.class)   // 这个还好
public DataSource ds(DruidDataSource raw) { }  // 这个会崩：方法签名要求类存在
```

**细节二：`@ConditionalOnMissingBean` 只能看见"已经处理过的" bean。**

它的判断依据是 `BeanDefinitionRegistry` 里**当前时刻**已注册的定义。所以：

- 同一个配置类里，方法 A 声明了 `@Bean Foo foo()`，方法 B 上有 `@ConditionalOnMissingBean(Foo.class)`——B 能否看到 A？**能**，因为同一个配置类内的方法按声明顺序处理，且 Spring 会先收集本类的 `@Bean` 方法。
- 跨配置类，就要靠 `@AutoConfigureAfter` / `@AutoConfigureBefore` 保证顺序。

**细节三：`@ConditionalOnProperty` 的 `havingValue` 语义。**

```java
@ConditionalOnProperty(prefix = "spring.redis", name = "enabled",
                       havingValue = "true", matchIfMissing = true)
```

- 配了 `spring.redis.enabled=true` → 成立
- 配了 `spring.redis.enabled=false` → 不成立
- **没配** → 看 `matchIfMissing`，默认 `false` 表示不成立

`matchIfMissing = true` 是"默认开"的意思，很多人漏了这个参数，结果"不配就不生效"，排查半天。

### 组合条件

三个注解可以把多个条件用"与/或/非"组合：

```java
// 全部成立才成立
static class OnBoth extends AllNestedConditions {
    OnBoth() { super(ConfigurationPhase.REGISTER_BEAN); }
    @ConditionalOnClass(name = "redis.clients.jedis.Jedis") static class HasJedis { }
    @ConditionalOnProperty("spring.redis.host") static class HasHost { }
}

// 任一成立即成立
static class OnEither extends AnyNestedCondition { ... }

// 全部不成立才成立
static class OnNeither extends NoneNestedConditions { ... }
```

`ConfigurationPhase` 有两个值：`PARSE_CONFIGURATION`（解析配置类时判断）和 `REGISTER_BEAN`（注册 bean 时判断）。默认是后者。区别在于——前者能看见的 bean 更少。

### 自定义 Condition 实战

需求：只在 `spring.profiles.active` 包含 `prod` **且** 存在 `application-prod.properties` 文件时才启用某配置。

```java
public class ProdResourceCondition implements Condition {
    @Override
    public boolean matches(ConditionContext ctx, AnnotatedTypeMetadata md) {
        String[] profiles = ctx.getEnvironment().getActiveProfiles();
        boolean isProd = Arrays.asList(profiles).contains("prod");
        boolean hasFile = ctx.getResourceLoader()
                .getResource("classpath:application-prod.properties").exists();
        return isProd && hasFile;
    }
}

@Configuration
@Conditional(ProdResourceCondition.class)
public class ProdOnlyConfig { }
```

### 用 --debug 看清条件

启动加参数：

```bash
java -jar app.jar --debug
```

会打印 `CONDITIONS EVALUATION REPORT`，分三块：

- **Positive matches** —— 哪些自动配置被装上了，因为什么条件成立
- **Negative matches** —— 哪些没装上，因为哪个条件不成立
- **Exclusions** —— 被显式排除的
- **Unconditional classes** —— 无条件装的

这是排查自动配置问题的**第一手段**，比翻源码快得多。
