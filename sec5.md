# 五、启动流程：SpringApplication.run 的十二步

《Spring 揭秘》讲的是 `refresh()` 内部的十二步。Spring Boot 的 `run()` 是**包在 `refresh()` 外面**的一层，一共十二步左右。

### 构造阶段：new SpringApplication()

```java
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    this.resourceLoader = resourceLoader;
    this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    this.webApplicationType = WebApplicationType.deduceFromClasspath();   // ① 推断应用类型
    this.bootstrapRegistryInitializers = getBootstrapRegistryInitializersFromSpringFactories();
    setInitializers(getSpringFactoriesInstances(ApplicationContextInitializer.class)); // ② 加载初始化器
    setListeners(getSpringFactoriesInstances(ApplicationListener.class));  // ③ 加载监听器
    this.mainApplicationClass = deduceMainApplicationClass();              // ④ 推断主类
}
```

**① 推断应用类型**，三种：

| 类型 | 判定条件 | 用哪个上下文 |
|---|---|---|
| `SERVLET` | classpath 有 `javax.servlet.Servlet` 且没有 `reactive.web.WebApplicationType` 的两个类 | `AnnotationConfigServletWebServerApplicationContext` |
| `REACTIVE` | 有 `DispatcherHandler` 且没有 `DispatcherServlet` | `AnnotationConfigReactiveWebServerApplicationContext` |
| `NONE` | 以上都不满足 | `AnnotationConfigApplicationContext` |

**②③ 加载 Initializer 和 Listener**，同样通过 `SpringFactoriesLoader` 读 `spring.factories`。这就是为什么你在 `spring.factories` 里注册一个 `ApplicationListener` 就能监听到启动事件。

**④ 推断主类**，靠遍历当前线程堆栈找 `main` 方法所在类。

### 运行阶段：run()

```java
public ConfigurableApplicationContext run(String... args) {
    StopWatch stopWatch = new StopWatch();          // ① 计时开始
    stopWatch.start();
    DefaultBootstrapContext bootstrapContext = createBootstrapContext();
    ConfigurableApplicationContext context = null;
    configureHeadlessProperty();                    // ② headless 模式
    SpringApplicationRunListeners listeners = getRunListeners(args);  // ③ 启动监听器
    listeners.starting(bootstrapContext, this.mainApplicationClass);
    try {
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
        ConfigurableEnvironment environment = prepareEnvironment(listeners, bootstrapContext, applicationArguments);  // ④ 准备环境
        configureIgnoreBeanInfo(environment);
        Banner printedBanner = printBanner(environment);   // ⑤ 打印 banner
        context = createApplicationContext();              // ⑥ 创建上下文
        context.setApplicationStartup(this.applicationStartup);
        prepareContext(bootstrapContext, context, environment, listeners, applicationArguments, printedBanner);  // ⑦ 准备上下文
        refreshContext(context);                           // ⑧ 刷新（核心！）
        afterRefresh(context, applicationArguments);       // ⑨ 刷新后
        stopWatch.stop();
        if (this.logStartupInfo) {
            new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
        }
        listeners.started(context, timeTakenToStartup);    // ⑩ 已启动事件
        callRunners(context, applicationArguments);        // ⑪ 调用 Runner
    } catch (Throwable ex) {
        handleRunFailure(context, ex, listeners);          // 异常处理
        throw new IllegalStateException(ex);
    }
    try {
        listeners.running(context);                        // ⑫ 运行中事件
    } catch (Throwable ex) { ... }
    return context;
}
```

### 重点看第 ⑧ 步：refreshContext

```java
private void refreshContext(ConfigurableApplicationContext context) {
    if (this.registerShutdownHook) {
        shutdownHook.registerApplicationContext(context);
    }
    refresh(context);   // 实际调用 AbstractApplicationContext.refresh()
}
```

**这里就是《Spring 揭秘》整本书讲的那十二步。**

`refresh()` 内部：

```
1  prepareRefresh()                → 记录启动时间、校验必需属性
2  obtainFreshBeanFactory()        → 创建 DefaultListableBeanFactory
3  prepareBeanFactory()            → 注册 ApplicationContextAwareProcessor 等
4  postProcessBeanFactory()        → 子类扩展（Web 容器在这里埋点）
5  invokeBeanFactoryPostProcessors()  ★ 自动配置在这里发生
6  registerBeanPostProcessors()    → 注册所有 BeanPostProcessor
7  initMessageSource()             → 国际化
8  initApplicationEventMulticaster() → 事件广播器
9  onRefresh()                     ★ 内嵌 Tomcat 在这里创建并启动
10 registerListeners()             → 注册监听器
11 finishBeanFactoryInitialization() ★ 所有单例 bean 在这里实例化
12 finishRefresh()                 → 发布 ContextRefreshedEvent
```

对照看：

- 第 5 步 —— 自动配置的 `ConfigurationClassPostProcessor` 在这里跑，把 100 多个候选筛成十几个，注册成 `BeanDefinition`。
- 第 9 步 —— `ServletWebServerApplicationContext.onRefresh()` 调用 `createWebServer()`，Tomcat 在这里 new 出来、bind 端口。**注意：此时 bean 还没实例化完，所以端口占用异常会在这时候抛，日志里看不到 "Started Application"。**
- 第 11 步 —— `getBean()` 真正造对象，`@PostConstruct`、`InitializingBean`、`AOP 代理`全在这一步。

::: tip 这是两本书的接缝
`SpringApplication.run()` 的 12 步里，第 8 步一进去就是《Spring 揭秘》的 12 步。**Spring Boot 在 `refresh()` 前面加的东西，都是"准备"；后面加的，都是"通知"。真正的装配，一点没变。**
:::

### 第 ⑦ 步 prepareContext 做了什么

```java
private void prepareContext(...) {
    context.setEnvironment(environment);
    postProcessApplicationContext(context);
    applyInitializers(context);              // 调用所有 ApplicationContextInitializer
    listeners.contextPrepared(context);      // 事件：上下文已准备
    // 注册 springApplicationArguments、springBootBanner 等特殊 bean
    // 把主配置类（@SpringBootApplication 那个）注册成 bean definition
    load(context, sources.toArray(new Object[0]));
    listeners.contextLoaded(context);        // 事件：配置类已加载
}
```

注意 `load()` 只是**注册**主配置类的 `BeanDefinition`，还没解析。解析发生在第 ⑧ 步的 `ConfigurationClassPostProcessor` 里。

### 第 ⑪ 步：Runner

`ApplicationRunner` 和 `CommandLineRunner` 在这里被调用，参数分别是 `ApplicationArguments` 和 `String[] args`。常用来做"启动后预热"，比如把商品库存刷进 Redis。

顺序问题：多个 Runner 用 `@Order` 或实现 `Ordered` 控制，数字小的先跑。

### 启动失败怎么排查

`handleRunFailure()` 会：

1. 发布 `ApplicationFailedEvent`
2. 调用 `SpringBootExceptionReporter` 分析异常（比如端口占用会给出友好提示）
3. 关闭上下文

想看完整的失败分析报告，加 `--debug`。想看 bean 依赖图，加 `--debug` 后搜 `Bean dependency graph`。
