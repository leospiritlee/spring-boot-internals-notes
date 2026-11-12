# Spring Boot 核心原理

> 不讲 API 怎么调，只讲"它凭什么能自己装配起来"。

## 序：从《Spring 揭秘》往前走一步

你刚读完《Spring 揭秘》。那本书讲的是 2009 年的 Spring 2.5：`applicationContext.xml` 里一行行 `<bean>`，一个 DAO 配一个 `DataSource`，一个 Service 配三个属性。书里花了整整一个部分讲 IoC 容器的 `refresh()` 十二步，讲 `BeanDefinition` 怎么被读进来、怎么被实例化、怎么被填充属性。

**那套东西今天一行都没变。** Spring Boot 没有重写 IoC 容器，`AbstractApplicationContext.refresh()` 还是那十二步，`DefaultListableBeanFactory` 还是那个工厂，`BeanPostProcessor` 还是那些扩展点。

变的是**谁来写那些 `<bean>`**。

以前是人。你要用 MyBatis，就得手写 `SqlSessionFactoryBean`、`MapperScannerConfigurer`；你要用 Redis，就得手写 `JedisPool`；你要用事务，就得配 `DataSourceTransactionManager` 再 `<tx:annotation-driven/>`。配置写得对不对，全看你记不记得住。

现在是**程序推导**。你只要把 jar 放进 classpath，Spring Boot 就会在启动时问自己一串问题：classpath 里有没有 `DataSource` 这个类？有没有配 `spring.datasource.url`？容器里是不是已经有人注册过同名的 bean？根据答案决定"这一坨 bean 要不要装"。

这个"问问题"的机制叫**条件装配**，是整个 Spring Boot 唯一的原创思想。其余都是工程包装。

### 这本书的定位

| | 《Spring 揭秘》 | 本书 |
|---|---|---|
| 关注 | 容器**内部**怎么运转 | 容器**外部**怎么被装配 |
| 核心问题 | `getBean()` 怎么造出一个对象 | 谁决定要造哪些对象 |
| 关键类 | `AbstractBeanFactory`、`BeanDefinition` | `AutoConfigurationImportSelector`、`Condition` |
| 时代 | Spring 2.5，XML | Spring Boot 2.x / 3.x，注解 + 约定 |

读完《Spring 揭秘》你知道了"bean 是怎么来的"；读完这本你要知道"bean 是谁决定要来的"。

::: warn 一句话先记住
Spring Boot = Spring Framework + 一套"按条件自动注册 BeanDefinition"的机制 + 一种新的打包运行方式。**没有第四样东西。**
:::