# 缓存

缓存是提升性能的一大利器，但缓存的处理，比如何时失效，如何失效等也是非常复杂的。在这一节我们一起来学习 Spring 的基于注释的 Cache 配置方法，展现了 Spring Cache 的强大之处，然后介绍了其基本的原理，扩展点和使用场景的限制。

Spring 从 `3.x` 开始引入了基于注解的缓存技术，它本质上是一个对缓存使用的抽象，通过在现有代码中添加注解，就可以够达到缓存方法的返回对象的效果。
Spring 的缓存具备相当的灵活性，不仅能够使用 `SpEL` （Spring Expression Language）来定义缓存的 key 和各种条件，也支持各自主流 Cache 框架的集成。其特点总结如下：

* 通过注解即可使得现有代码支持缓存
* 支持开箱即用，即不用安装和部署额外第三方组件就可以使用缓存
* 支持 SpEL 表达式，能使用对象的任何属性或者方法来定义缓存的键值和条件
* 支持 AOP
* 支持自定义键值和自定义 CacheManager ，具有相当的灵活性和扩展性

## 配置 Cache

最简化的配置其实只需要添加 `@EnableCaching` 即可，也就是我们可以在 `Application.java` 的类上面加这行注解就可以让应用支持 Cache 了。当然我们后面如果要做一些定制化的话，还是新建一个配置文件，在 `config` 包下新建 `CacheConfig.java`

```java
package dev.local.gtm.api.config;

import org.springframework.cache.annotation.EnableCaching;
import org.springframework.context.annotation.Configuration;

@EnableCaching
@Configuration
public class CacheConfig {
}
```

是的，就是这么简单，在我们没有指定其他的 Cache 实现方式之前，Spring Boot 会自动帮我们配置一个简单的 Cache 实现。这个实现方式是使用 `ConcurrentHashMap` 作为缓存的存储。默认情况下，系统会在需要的时候创建缓存，但你也可以限定缓存的数量，这可以通过设置 `cache-names` 来实现。如果你使用了这个列表之外的缓存，Spring Boot 应用会无法启动。

```yml
spring:
  cache:
    cache-names: usersByLogin, usersByMobile, usersByEmail
```

除了这种默认的简单缓存之外， Spring 支持以下类型的缓存框架

* Generic
* JCache (JSR-107)
* EhCache 2.x
* Hazelcast
* Infinispan
* Couchbase
* Redis
* Caffeine

为了让我们可以后期看到 Cache 是否生效了，我们可以配置 Cache 的日志，更改 `application.yml`  设置 `dev` 环境下的 `org.springframework.cache=DEBUG`

```yml
spring:
  application:
    name: api-service
  profiles:
    active: dev
  data:
    mongodb:
      database: gtm-api
  cache:
    type: redis
  redis:
    host: redis
server:
  port: 8080
logging:
  level:
    org.apache.http: ERROR
    org.springframework:
      web: ERROR
      data: ERROR
      security: ERROR
      cache: ERROR
    org.springframework.data.mongodb.core.MongoTemplate: ERROR
    dev.local.gtm.api: ERROR

---

spring:
  profiles: dev
  devtools:
    remote:
      secret: thisismysecret
  data:
    mongodb:
      database: gtm-api-dev
  redis:
    host: localhost
logging:
  level:
    org.apache.http: DEBUG
    org.springframework:
        web:
          client.RestTemplate: DEBUG
        data: DEBUG
        security: DEBUG
        cache: DEBUG
    org.springframework.data.mongodb.core.MongoTemplate: DEBUG
    dev.local.gtm.api: DEBUG

---

spring:
  profiles: test
  data:
    mongodb:
      database: gtm-api-test
  cache:
    type: none
logging:
  level:
    org.apache.http: DEBUG
    org.springframework:
          web:
            client.RestTemplate: DEBUG
          data: DEBUG
          security: DEBUG
    org.springframework.data.mongodb.core.MongoTemplate: DEBUG
    dev.local.gtm.api: DEBUG
---

spring:
  profiles: prod
```

## 常用的缓存注解

还记得我们在 `UserRepo` 中曾经设置过几个注解吗？ `@Cacheable(cacheNames = xxx)`

```java
@Repository
public interface UserRepo extends MongoRepository<User, String> {
    String USERS_BY_LOGIN_CACHE = "usersByLogin";

    String USERS_BY_MOBILE_CACHE = "usersByMobile";

    String USERS_BY_EMAIL_CACHE = "usersByEmail";

    @Cacheable(cacheNames = USERS_BY_MOBILE_CACHE)
    Optional<User> findOneByMobile(@Param("mobile") String mobile);

    @Cacheable(cacheNames = USERS_BY_EMAIL_CACHE)
    Optional<User> findOneByEmailIgnoreCase(@Param("email") String email);

    @Cacheable(cacheNames = USERS_BY_LOGIN_CACHE)
    Optional<User> findOneByLogin(@Param("login") String login);

    Page<User> findAllByLoginNot(Pageable pageable, @Param("login") String login);
}
```

Spring 中提供以下几个注解

* `@Cacheable` 触发缓存triggers cache population
* `@CacheEvict` 触发缓存的回收
* `@CachePut` 更新缓存
* `@Caching` 一组应用到某个方法上的缓存
* `@CacheConfig` 在类级别上共享一些缓存相关的设置

### `@Cacheable`

顾名思义， `@Cacheable` 是用来标记可以缓存的方法，也就是说方法的返回值会存储在缓存中，下次如果以同样的参数访问这个方法时，我们会返回缓存中的值而不是又一次执行这个方法。 我们在 `UserRepo` 中使用的就是 `@Cacheable` 的最简化形式

```java
@Cacheable(cacheNames = USERS_BY_MOBILE_CACHE)
Optional<User> findOneByMobile(@Param("mobile") String mobile);
```

这个例子中， `findOneByMobile` 和名字为 `usersByMobile` ( `String USERS_BY_MOBILE_CACHE = "usersByMobile";` ) 的缓存关联了。每次这个方法被调用时，缓存会检查这个方法是否已经调用过，如果调用过，就直接返回缓存值。

#### 默认键值策略

由于缓存很多时候是键值对（ `key-value` ），所以每一次方法的调用需要映射成一个 `key` ，以便我们可以通过这个 `key` 访问缓存中的值。 Spring 的默认 `key` 生成是通过一个简单的 `KeyGenerator` ，遵循以下规则

* 如果没有参数，那么返回 `SimpleKey.EMPTY` 。
* 如果只有一个参数，返回这个参数的实例。
* 二个及以上的参数，返回一个包含所有参数的 SimpleKey ，参数需要实现 `equals()` 和 `hashCode()` 方法避免 `key` 值重复。

有的时候，对于缓存来说，一个方法的参数并不是都有同等重要性。如果参数只是在方法中使用，而不是影响结果，那么它可能不适合作为 `Cache` 的 `key` 。比如下面的例子中， `id` 作为唯一识别文件的标识，我们指定其做为 `key` ，而另一个参数不参与（仅仅作为示例，实际的商业逻辑不一定是这样的，请勿照搬）。

```java
@Cacheable(cacheNames="files", key="#id")
public String findFile(String id, boolean includeArchived)
```

当然这个 `key` 可以使用 SpEL 表达式，比如 `key="T(someparam).hash(#someparam)` 等，也可以指定自定义 `key` 生成器，比如 `@Cacheable(cacheNames="files", keyGenerator="customKeyGenerator")` ，这可以通过实现 `org.springframework.cache.interceptor.KeyGenerator` 接口来自定义。

## 测试缓存是否生效

如果我们在 API 文档界面 <http://localhost:8080/swagger-ui/index.html> 对 `/api/auth/search/username` 进行测试的话，我们可以观察 `Console` 里面的日志

![利用 API 文档界面进行测试](/assets/2018-04-27-19-32-26.png)

在 Spring Boot 启动的日志中可以看到类似下面这种日志，这是 Cache 配置成功的标志，系统 Cache 框架将我们注解标记的方法缓存起来了。

```log
  : Adding cacheable method 'findOneByEmailIgnoreCase' with attribute: [Builder[public abstract j
ava.util.Optional dev.local.gtm.api.repository.UserRepo.findOneByEmailIgnoreCase(java.lang.String)] caches=[usersByEmail] | key='' | keyGenerator='' | cacheManager='' | cacheResolver='' | condition='' | unless='' | sync='false']
2018-04-27 19:50:20.207 DEBUG 10821 --- [           main] o.s.c.a.AnnotationCacheOperationSource
  : Adding cacheable method 'emailExisted' with attribute: [Builder[public dev.local.gtm.api.web.
rest.AuthResource$ExistCheck dev.local.gtm.api.web.rest.AuthResource.emailExisted(java.lang.String)] caches=[usersByEmail] | key='' | keyGenerator='' | cacheManager='' | cacheResolver='' | condition='' | unless='' | sync='false']
```

我们可以观察到日志中，有 `AuthServiceImpl.usernameExisted()` 方法进入的条目，也有 `MongoTemplate` 进行数据库查询的条目，这证明第一次进行查询的时候，没有进行缓存，这是正常情况。

```log
2018-04-27 19:29:41.981 DEBUG 9561 --- [ XNIO-2 task-19] d.l.gtm.api.aop.logging.LoggingAspect : Enter: dev.local.gtm.api.web.rest.AuthResource.usernameExisted() with argument[s] = [test123]
2018-04-27 19:29:41.981 DEBUG 9561 --- [ XNIO-2 task-19] dev.local.gtm.api.web.rest.AuthResource : REST 请求 -- 用户名是否存在 test123
2018-04-27 19:29:41.982 DEBUG 9561 --- [ XNIO-2 task-19] d.l.gtm.api.aop.logging.LoggingAspect : Enter: dev.local.gtm.api.service.impl.AuthServiceImpl.usernameExisted() with argument[s] = [test123]
2018-04-27 19:29:41.988 DEBUG 9561 --- [ XNIO-2 task-19] o.s.d.m.r.query.MongoQueryCreator : Created query Query: { "login" : "test123" }, Fields: { }, Sort: { }
2018-04-27 19:29:41.990 DEBUG 9561 --- [ XNIO-2 task-19] o.s.data.mongodb.core.MongoTemplate : find using query: { "login" : "test123" } fields: Document{{}} for class: class dev.local.gtm.api.domain.User in collection: api_users
2018-04-27 19:29:41.996 DEBUG 9561 --- [ XNIO-2 task-19] d.l.gtm.api.aop.logging.LoggingAspect : Exit: dev.local.gtm.api.service.impl.AuthServiceImpl.usernameExisted() with result = false
2018-04-27 19:29:41.997 DEBUG 9561 --- [ XNIO-2 task-19] d.l.gtm.api.aop.logging.LoggingAspect : Exit: dev.local.gtm.api.web.rest.AuthResource.usernameExisted() with result = dev.local.gtm.api.web.rest.AuthResource$ExistCheck@32ef7921
2018-04-27 19:29:41.999 DEBUG 9561 --- [ XNIO-2 task-19] o.s.s.w.header.writers.HstsHeaderWriter : Not injecting HSTS header since it did not match the requestMatcher org.springframework.security.web.header.writers.HstsHeaderWriter$SecureRequestMatcher@e228328
2018-04-27 19:29:42.002 DEBUG 9561 --- [ XNIO-2 task-19] o.s.s.w.a.ExceptionTranslationFilter : Chain processed normally
2018-04-27 19:29:42.002 DEBUG 9561 --- [ XNIO-2 task-19] s.s.w.c.SecurityContextPersistenceFilter : SecurityContextHolder now cleared, as request processing completed
```

接下来，我们再次执行同样的请求，还是观察日志输出，这一次我们找不到任何 MongoDB 查询的日志，

```log
2018-04-27 19:42:22.733 DEBUG 9561 --- [ XNIO-2 task-20] d.l.gtm.api.aop.logging.LoggingAspect : Enter: dev.local.gtm.api.web.rest.AuthResource.usernameExisted() with argument[s] = [test123]
2018-04-27 19:42:22.733 DEBUG 9561 --- [ XNIO-2 task-20] dev.local.gtm.api.web.rest.AuthResource : REST 请求 -- 用户名是否存在 test123
2018-04-27 19:42:22.733 DEBUG 9561 --- [ XNIO-2 task-20] d.l.gtm.api.aop.logging.LoggingAspect : Enter: dev.local.gtm.api.service.impl.AuthServiceImpl.usernameExisted() with argument[s] = [test123]
2018-04-27 19:42:22.758 DEBUG 9561 --- [ XNIO-2 task-20] d.l.gtm.api.aop.logging.LoggingAspect
 : Exit: dev.local.gtm.api.service.impl.AuthServiceImpl.usernameExisted() with result = false
2018-04-27 19:42:22.758 DEBUG 9561 --- [ XNIO-2 task-20] d.l.gtm.api.aop.logging.LoggingAspect : Exit: dev.local.gtm.api.web.rest.AuthResource.usernameExisted() with result = dev.local.gtm.api.web.rest.AuthResource$ExistCheck@53c05c2
2018-04-27 19:42:22.766 DEBUG 9561 --- [ XNIO-2 task-20] o.s.s.w.header.writers.HstsHeaderWriter : Not injecting HSTS header since it did not match the requestMatcher org.springframework.security.web.header.writers.HstsHeaderWriter$SecureRequestMatcher@e228328
2018-04-27 19:42:22.773 DEBUG 9561 --- [ XNIO-2 task-20] o.s.s.w.a.ExceptionTranslationFilter : Chain processed normally
2018-04-27 19:42:22.773 DEBUG 9561 --- [ XNIO-2 task-20] s.s.w.c.SecurityContextPersistenceFilter : SecurityContextHolder now cleared, as request processing completed
```
