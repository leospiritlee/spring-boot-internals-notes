# 九、Actuator：让运行中的应用可观测

### 加依赖

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

### 默认暴露的端点

只有 `/actuator/health` 和 `/actuator/info`。其余要显式打开：

```properties
management.endpoints.web.exposure.include=health,info,metrics,env,beans,threaddump,loggers
management.endpoint.health.show-details=when-authorized
```

### 常用端点

| 端点 | 用途 | 生产建议 |
|---|---|---|
| `/health` | 健康检查，聚合 DB / Redis / MQ / 磁盘 | 必须开，给 LB 用 |
| `/info` | 版本、构建信息 | 开 |
| `/metrics` | Micrometer 指标 | 开，但要限流 |
| `/env` | 所有配置项及来源 | **别对外暴露**，会泄露密码 |
| `/beans` | 所有 bean 及其依赖 | 排障时临时开 |
| `/threaddump` | 线程栈 | 排障时临时开 |
| `/heapdump` | 堆转储 | **绝对别暴露** |
| `/loggers` | 运行时改日志级别 | 开，很好用 |
| `/mappings` | 所有 URL 映射 | 排障时开 |
| `/caches` | 缓存状态 | 按需 |
| `/shutdown` | 优雅停机 | 默认关闭，要开得配 `management.endpoint.shutdown.enabled=true` |

::: warn 生产环境的暴露面
`/env` 和 `/heapdump` 是事故高发点。前者会把你所有配置（含数据库密码）以明文返回，后者能让攻击者下载整个堆内存。

最低限度的保护：

```properties
management.endpoints.web.exposure.include=health,info
management.endpoint.health.show-details=never
management.server.port=8081          # 管理端口与应用端口分离
```

如果管理端口不能对外，用 `management.server.address=127.0.0.1` 只监听本机，再用跳板机访问。
:::

### 自定义健康检查

```java
@Component
public class RedisHealthIndicator implements HealthIndicator {
    private final JedisPool pool;

    public RedisHealthIndicator(JedisPool pool) { this.pool = pool; }

    @Override
    public Health health() {
        try (Jedis jedis = pool.getResource()) {
            String pong = jedis.ping();
            return "PONG".equalsIgnoreCase(pong)
                    ? Health.up().withDetail("latency", "ok").build()
                    : Health.down().withDetail("ping", pong).build();
        } catch (Exception e) {
            return Health.down(e).build();
        }
    }
}
```

命名规则：bean 名去掉 `HealthIndicator` 后缀，就是 `/health` 里的 key。上面这个会出现在 `health.components.redis`。

### 自定义指标

```java
@Component
public class SeckillMetrics {
    private final Counter successCounter;
    private final Timer costTimer;

    public SeckillMetrics(MeterRegistry registry) {
        this.successCounter = Counter.builder("seckill.success")
                .description("秒杀成功次数").register(registry);
        this.costTimer = Timer.builder("seckill.cost")
                .description("秒杀接口耗时").register(registry);
    }

    public void record(Runnable r) {
        costTimer.record(() -> { r.run(); successCounter.increment(); });
    }
}
```

Micrometer 是门面，底层可以对接 Prometheus、Datadog、InfluxDB。接 Prometheus 只需加 `micrometer-registry-prometheus` 依赖，然后 `/actuator/prometheus` 就有标准格式的指标。
