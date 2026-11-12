# 八、手写一个 starter

看懂自动配置之后，写 starter 就是照葫芦画瓢。这一章从零做一个"请求耗时统计" starter。

### 目标

引入 `@EnableCostTime` 或直接引 jar，就能自动统计所有 Controller 方法的耗时，打印慢请求。可配置阈值，可关闭。

### 目录结构

```
cost-time-spring-boot-starter/
├── pom.xml
└── src/main/
    ├── java/com/example/costtime/
    │   ├── CostTimeProperties.java
    │   ├── CostTimeInterceptor.java
    │   ├── CostTimeAutoConfiguration.java
    │   └── EnableCostTime.java
    └── resources/
        └── META-INF/
            ├── spring.factories                                    ← 2.x
            └── spring/
                └── org.springframework.boot.autoconfigure.AutoConfiguration.imports  ← 2.7+/3.x
```

### 一、配置属性

```java
@ConfigurationProperties(prefix = "cost-time")
public class CostTimeProperties {
    /** 是否启用 */
    private boolean enabled = true;
    /** 超过多少毫秒算慢请求 */
    private long slowThreshold = 500;
    /** 慢请求日志前缀 */
    private String logPrefix = "[SLOW]";

    // getter / setter 省略
}
```

注意**没有 `@Component`**。starter 里的配置类不应该依赖使用者的包扫描。

### 二、核心逻辑

```java
public class CostTimeInterceptor implements HandlerInterceptor {
    private final CostTimeProperties props;
    private static final String START = "__cost_start__";

    public CostTimeInterceptor(CostTimeProperties props) {
        this.props = props;
    }

    @Override
    public boolean preHandle(HttpServletRequest req, HttpServletResponse resp, Object handler) {
        req.setAttribute(START, System.nanoTime());
        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest req, HttpServletResponse resp,
                                Object handler, Exception ex) {
        Long start = (Long) req.getAttribute(START);
        if (start == null) return;
        long costMs = (System.nanoTime() - start) / 1_000_000;
        if (costMs >= props.getSlowThreshold()) {
            System.out.printf("%s %s %s 耗时 %d ms%n",
                    props.getLogPrefix(), req.getMethod(), req.getRequestURI(), costMs);
        }
    }
}
```

### 三、自动配置类

```java
@AutoConfiguration
@ConditionalOnWebApplication(type = ConditionalOnWebApplication.Type.SERVLET)
@ConditionalOnProperty(prefix = "cost-time", name = "enabled",
                       havingValue = "true", matchIfMissing = true)
@EnableConfigurationProperties(CostTimeProperties.class)
public class CostTimeAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public CostTimeInterceptor costTimeInterceptor(CostTimeProperties props) {
        return new CostTimeInterceptor(props);
    }

    @Bean
    @ConditionalOnMissingBean(name = "costTimeWebMvcConfigurer")
    public WebMvcConfigurer costTimeWebMvcConfigurer(CostTimeInterceptor interceptor) {
        return new WebMvcConfigurer() {
            @Override
            public void addInterceptors(InterceptorRegistry registry) {
                registry.addInterceptor(interceptor).addPathPatterns("/**");
            }
        };
    }
}
```

三个注解各自的作用：

- `@ConditionalOnWebApplication(type = SERVLET)` —— 非 Web 应用（比如批处理）不装。
- `@ConditionalOnProperty(..., matchIfMissing = true)` —— 默认开启，配 `cost-time.enabled=false` 可关。
- `@ConditionalOnMissingBean` —— 使用者自己定义了 `CostTimeInterceptor` 就用他的，不覆盖。

`@AutoConfiguration` 是 2.7 引入的，等价于 `@Configuration(proxyBeanMethods = false)` 加 `@AutoConfigureBefore/After` 的语义。2.x 老项目用 `@Configuration` 也一样。

### 四、注册清单

**方式 A：spring.factories（2.x）**

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.example.costtime.CostTimeAutoConfiguration
```

**方式 B：AutoConfiguration.imports（2.7+ / 3.x）**

文件名：`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`

```
com.example.costtime.CostTimeAutoConfiguration
```

一行一个类，不用写 key，不用反斜杠续行。

::: warn 3.x 只认方式 B
Spring Boot 3.0 起，`spring.factories` 里的 `EnableAutoConfiguration` 条目**被彻底忽略**。只放 A 的 starter 在 3.x 里会"引了但没生效"，而且不报错。兼容做法是两个文件都放。
:::

### 五、pom.xml 的关键点

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-configuration-processor</artifactId>
  <optional>true</optional>
</dependency>
```

`spring-boot-configuration-processor` 是注解处理器，编译时扫描 `@ConfigurationProperties` 类，生成 `META-INF/spring-configuration-metadata.json`。有了它，使用者在 IDE 里敲 `cost-time.` 就能看到补全和注释。

`<optional>true</optional>` 表示这个依赖不传递给使用者——它只在编译期有用。

### 六、测试 starter 的自动配置

```java
@SpringBootTest
class CostTimeAutoConfigurationTest {
    @Autowired
    ApplicationContext ctx;

    @Test
    void interceptor_should_be_registered() {
        assertThat(ctx.getBeansOfType(CostTimeInterceptor.class)).hasSize(1);
    }
}
```

更严格的测法是 `ApplicationContextRunner`：

```java
private final ApplicationContextRunner runner = new ApplicationContextRunner()
        .withConfiguration(AutoConfigurations.of(CostTimeAutoConfiguration.class));

@Test
void disabled_when_property_false() {
    runner.withPropertyValues("cost-time.enabled=false")
          .run(ctx -> assertThat(ctx).doesNotHaveBean(CostTimeInterceptor.class));
}

@Test
void backed_off_when_user_defines_own() {
    runner.withUserConfiguration(UserConfig.class)
          .run(ctx -> assertThat(ctx.getBean(CostTimeInterceptor.class))
                  .isSameAs(UserConfig.MY_INSTANCE));
}
```

`ApplicationContextRunner` 是测自动配置的标准工具，比 `@SpringBootTest` 快得多——它不启整个应用，只跑你指定的配置类。
