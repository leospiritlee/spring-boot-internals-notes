# 三、自动配置全链路

这是全书最核心的一章。看懂这条链，Spring Boot 对你就不再是黑盒。

### 起点：@EnableAutoConfiguration

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration { }
```

关键是 `@Import(AutoConfigurationImportSelector.class)`。`@Import` 的语义是"把这个类当成配置类处理"，而 `ImportSelector` 是一个**能动态返回一批类名**的扩展点：

```java
public interface ImportSelector {
    String[] selectImports(AnnotationMetadata importingClassMetadata);
}
```

返回的每个类名都会被当成 `@Configuration` 类去解析。**所以自动配置的本质是：启动时动态生成一份"配置类清单"，交给容器去解析。**

### 第二步：收集候选

`AutoConfigurationImportSelector.selectImports()` 内部调用 `getCandidateConfigurations()`，最终落到：

```java
List<String> configurations = SpringFactoriesLoader.loadFactoryNames(
        EnableAutoConfiguration.class, classLoader);
```

`SpringFactoriesLoader` 干的事很朴素：扫描 classpath 上**所有** jar 里的 `META-INF/spring.factories` 文件，把 `org.springframework.boot.autoconfigure.EnableAutoConfiguration=` 后面那一长串类名全读出来，合并成一个列表。

`spring-boot-autoconfigure` 这个 jar 里的 `spring.factories` 有 100 多行，对应 100 多个自动配置类：

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration,\
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\
org.springframework.boot.autoconfigure.cache.CacheAutoConfiguration,\
org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration,\
...
```

::: tip 2.7 / 3.0 的变化
Spring Boot 2.7 起新增 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 文件，一行一个类名，替代 `spring.factories` 里那一大坨。3.0 起 `spring.factories` 里的自动配置条目**不再被读取**，但 `spring.factories` 机制本身还留着给别的扩展点用（比如 `ApplicationListener`、`EnvironmentPostProcessor`）。

如果你写 starter 要兼容 2.x 和 3.x，两个文件都得放。
:::

### 第三步：过滤

收集到 100 多个候选，但一次启动通常只用到十几个。剩下的靠**条件**筛掉。

筛选发生在 `AutoConfigurationImportSelector.filter()`：

1. **先做一次粗筛**（`AutoConfigurationImportFilter`）。这一步是为了性能——很多自动配置类引用了 classpath 上不存在的类，如果直接加载它们的字节码会抛 `NoClassDefFoundError`。所以 Spring Boot 在编译期生成了一个 `META-INF/spring-autoconfigure-metadata.properties`：

   ```properties
   org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration.ConditionalOnClass=javax.sql.DataSource
   org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration.ConditionalOnClass=org.springframework.data.redis.core.RedisOperations
   ```

   用 ASM 读注解元数据（不加载类），比对 `@ConditionalOnClass` 里的类在不在 classpath，不在就直接踢掉。

2. **再做精确筛选**（`OnClassCondition` 之外的所有条件）。这一步才真正解析每个 `@Configuration` 类上的条件注解。

3. **最后去重 + 排除**。`@EnableAutoConfiguration(exclude=...)` 和 `spring.autoconfigure.exclude` 属性在这里生效。

### 第四步：排序

自动配置类之间有依赖顺序。比如 `DataSourceAutoConfiguration` 必须在 `MybatisAutoConfiguration` 之前。排序依据是 `@AutoConfigureBefore` / `@AutoConfigureAfter` / `@AutoConfigureOrder`，最后由 `AutoConfigurationSorter` 做拓扑排序。

::: warn 顺序错了会怎样
最常见的现象是 `@ConditionalOnMissingBean` 判断失误：A 想覆盖 B 的 bean，但 A 比 B 先执行，此时容器里还没有 B 的 bean，`OnMissingBean` 判定为"缺"，A 注册了；随后 B 执行，发现已经有 bean 了，B 跳过。结果跟预期相反。

这类问题不报错，只是行为不对，只能靠 `--debug` 看 `CONDITIONS EVALUATION REPORT` 排查。
:::

### 第五步：注册

筛选完的类名列表返回给 `ConfigurationClassParser`，走的是《Spring 揭秘》里讲的那套流程：解析成 `ConfigurationClass` → 注册 `BeanDefinition` → `refresh()` 里实例化。

**到这里，自动配置就结束了。** 它没有创造新的容器机制，只是在"配置类从哪来"这一步做了文章。

### 完整链路串一遍

```
SpringApplication.run()
  └─ refreshContext(context)
       └─ AbstractApplicationContext.refresh()
            └─ invokeBeanFactoryPostProcessors()
                 └─ ConfigurationClassPostProcessor
                      └─ ConfigurationClassParser.parse()
                           └─ 遇到 @Import(AutoConfigurationImportSelector)
                                └─ selectImports()
                                     ├─ SpringFactoriesLoader.loadFactoryNames()   ← 读清单
                                     ├─ removeDuplicates()                          ← 去重
                                     ├─ getExclusions()                             ← 排除
                                     ├─ filter()                                    ← 条件筛选
                                     │    ├─ AutoConfigurationImportFilter（粗筛，ASM）
                                     │    └─ ConditionEvaluator（精筛，真条件）
                                     └─ AutoConfigurationSorter.getInPriorityOrder() ← 排序
                           └─ 逐个 processConfigurationClass() → 注册 BeanDefinition
```

### 一个具体例子：DataSource 怎么被装上

`DataSourceAutoConfiguration` 的关键代码：

```java
@AutoConfiguration
@ConditionalOnClass({ DataSource.class, EmbeddedDatabaseType.class })
@EnableConfigurationProperties(DataSourceProperties.class)
@Import({ DataSourcePoolMetadataProvidersConfiguration.class,
          DataSourceInitializationConfiguration.class })
public class DataSourceAutoConfiguration {

    @Configuration(proxyBeanMethods = false)
    @Conditional(PooledDataSourceCondition.class)
    @ConditionalOnMissingBean({ DataSource.class, XADataSource.class })
    @Import({ DataSourceConfiguration.Hikari.class, ... })
    protected static class PooledDataSourceConfiguration { }
}
```

逐条翻译成人话：

- `@ConditionalOnClass(DataSource.class)` —— classpath 上有 JDBC 的接口才继续（引了 `spring-boot-starter-jdbc` 才有）。
- `@EnableConfigurationProperties(DataSourceProperties.class)` —— 把 `spring.datasource.*` 绑到一个对象上。
- `@ConditionalOnMissingBean(DataSource.class)` —— 你**没**自己定义 `DataSource` 才装。你要是写了 `@Bean DataSource myDs()`，这里就跳过，以你的为准。
- `@Import(DataSourceConfiguration.Hikari.class)` —— 真正造 Hikari 连接池的地方。

**注意最后一条。** 秒杀项目里引了 `druid-spring-boot-starter` 并自己配了 `DruidDataSource`，于是 `@ConditionalOnMissingBean` 判定为"已有"，Hikari 那套完全不生效。这就是"自动配置让位给显式配置"的机制。

::: tip 这一条是理解 Spring Boot 的钥匙
**自动配置永远是"兜底"，不是"覆盖"。** 你自己定义的 bean 优先级永远高于自动配置。所以排查"为什么我的配置没生效"，先看是不是被某个 `@ConditionalOnMissingBean` 挡掉了，而不是去改自动配置类。
:::
