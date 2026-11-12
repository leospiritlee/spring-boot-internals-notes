# 二、起步依赖：starter 不是魔法

### starter 里到底有什么

以 `spring-boot-starter-web` 为例，解开 jar 看：

```
spring-boot-starter-web-2.6.2.jar
└── META-INF/
    ├── MANIFEST.MF
    └── maven/...          # 只有元数据，没有 .class
```

**一个 class 都没有。** 它的全部价值在 `pom.xml`：

```xml
<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-json</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-tomcat</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-web</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-webmvc</artifactId>
  </dependency>
</dependencies>
```

注意：**一个 `<version>` 都没写**。

### 版本从哪来

父 POM 里 `spring-boot-starter-parent` 继承自 `spring-boot-dependencies`，后者是一个巨大的 `<dependencyManagement>`，里面锁死了几百个常用库的版本：

```xml
<parent>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-parent</artifactId>
  <version>2.6.2</version>
</parent>
```

`dependencyManagement` 的语义是"**如果你引了这个依赖但没写版本，就用我这里定的**"。所以 starter 里可以全部省略版本，而版本之间是官方测试过的兼容组合。

::: warn 别在子模块里随便写 version
一旦你在 `pom.xml` 里给某个 Spring 相关依赖显式写了版本，就会覆盖 BOM 的仲裁。常见后果是 Spring 5 的 `spring-core` 配 Spring 6 的 `spring-web`，启动时报 `NoSuchMethodError`，堆栈还指向框架内部，极难定位。
:::

### 官方 starter 与第三方 starter 的命名

| 类型 | 命名 | 例子 |
|---|---|---|
| 官方 | `spring-boot-starter-xxx` | `spring-boot-starter-data-redis` |
| 第三方 | `xxx-spring-boot-starter` | `mybatis-spring-boot-starter`、`druid-spring-boot-starter` |

这不是强制规范，但是社区共识。看到 `spring-boot-starter-` 开头就知道是官方的。

### starter 冲突怎么处理

两个 starter 带进来同一个库的不同版本，或者带进来功能重叠的东西（比如同时有 `spring-boot-starter-web` 和 `spring-boot-starter-webflux`，两个都想当 Web 容器）。

排查命令：

```bash
mvn dependency:tree -Dincludes=org.springframework:spring-core
```

排除写法：

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
  <exclusions>
    <exclusion>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-tomcat</artifactId>
    </exclusion>
  </exclusions>
</dependency>
```

上面这段是"换成 Undertow"的标准写法：先排掉 Tomcat，再单独引 `spring-boot-starter-undertow`。
