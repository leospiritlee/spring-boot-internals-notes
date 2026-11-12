# 七、内嵌容器与 fat jar

### Tomcat 是怎么被启动的

关键在 `ServletWebServerApplicationContext`，它重写了 `onRefresh()`：

```java
@Override
protected void onRefresh() {
    super.onRefresh();
    try {
        createWebServer();     // ← 就在这里把 Tomcat 拉起来
    } catch (Throwable ex) {
        throw new ApplicationContextException("Unable to start web server", ex);
    }
}

private void createWebServer() {
    WebServer webServer = this.webServer;
    ServletContext servletContext = getServletContext();
    if (webServer == null && servletContext == null) {
        ServletWebServerFactory factory = getWebServerFactory();
        this.webServer = factory.getWebServer(getSelfInitializer());
        getBeanFactory().registerSingleton("webServerGracefulShutdown",
                new WebServerGracefulShutdownLifecycle(this.webServer));
        getBeanFactory().registerSingleton("webServerStartStop",
                new WebServerStartStopLifecycle(this));
    }
}
```

`getWebServerFactory()` 从容器里找一个 `ServletWebServerFactory` 类型的 bean。`ServletWebServerFactoryAutoConfiguration` 会根据 classpath 决定装哪个：

| classpath 上有 | 装的工厂 |
|---|---|
| Tomcat（`spring-boot-starter-tomcat`） | `TomcatServletWebServerFactory` |
| Jetty | `JettyServletWebServerFactory` |
| Undertow | `UndertowServletWebServerFactory` |

三者都在时按 `@ConditionalOnClass` 的顺序，Tomcat 优先。

真正的端口绑定发生在 `WebServerStartStopLifecycle.start()`，由 `Lifecycle` 机制在 `finishRefresh()` 之后统一启动。

### fat jar 的结构

```
app.jar
├── META-INF/
│   └── MANIFEST.MF
├── org/springframework/boot/loader/
│   ├── JarLauncher.class
│   ├── LaunchedURLClassLoader.class
│   └── ...
├── BOOT-INF/
│   ├── classes/            ← 你的 class
│   │   └── com/example/...
│   └── lib/                ← 所有依赖 jar
│       ├── spring-core-5.3.14.jar
│       ├── tomcat-embed-core-9.0.56.jar
│       └── ...
└── (无)
```

`MANIFEST.MF` 里：

```
Main-Class: org.springframework.boot.loader.JarLauncher
Start-Class: com.example.Application
```

JVM 启动的是 `JarLauncher`（Spring Boot 自己写的类，**不是**你的主类）。它干三件事：

1. 创建 `LaunchedURLClassLoader`，把 `BOOT-INF/classes/` 和 `BOOT-INF/lib/*.jar` 都变成可加载的 URL。
2. 用这个类加载器加载 `Start-Class`。
3. 反射调用它的 `main` 方法。

### 为什么不用标准 jar

标准 jar 里不能再嵌 jar（`BOOT-INF/lib/` 里的那些 jar 是**原封不动**塞进去的，不是解压后的 class）。标准 `URLClassLoader` 不认识 `jar:file:app.jar!/BOOT-INF/lib/xxx.jar!/` 这种嵌套 URL，所以 Spring Boot 自己实现了 `LaunchedURLClassLoader` 和一套 `URLStreamHandler`。

::: warn fat jar 的坑
**坑一：`getResource` 找不到文件。** 在 fat jar 里，`ClassLoader.getResource("config/foo.txt")` 能拿到，但 `new File(url.getPath())` 一定失败，因为路径是 `jar:file:...!/config/foo.txt` 这种形式，不是真实文件系统路径。要么用 `getResourceAsStream`，要么用 `PathMatchingResourcePatternResolver`。

**坑二：`@ComponentScan` 扫不到。** 如果你把某个模块打成独立 jar 放在 `BOOT-INF/lib`，它的包不在主应用包下，默认扫不到。要显式 `@ComponentScan("com.other")` 或写 `spring.factories`。

**坑三：热部署失效。** `spring-boot-devtools` 靠重启类加载器实现热部署，但它只重启 `RestartClassLoader`（加载 `BOOT-INF/classes`），`BOOT-INF/lib` 里的 jar 由父加载器持有，改了不生效。
:::

### war 部署还留着的场景

| 场景 | 选择 |
|---|---|
| 公司统一用外部 Tomcat / WebLogic | war |
| 需要多个应用共享一个容器 | war |
| 云原生、容器化 | fat jar |
| 需要 `jsp` 支持（受限） | war 更省事 |

打成 war 只需两步：`<packaging>war</packaging>`，把 `spring-boot-starter-tomcat` 改成 `provided`，主类继承 `SpringBootServletInitializer` 重写 `configure()`。
