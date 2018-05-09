# Redis 作为缓存框架

Redis 是一个开源的，基于内存的基于键值对的数据库，是 NoSQL 的一种。一般用作数据库、缓存服务或消息服务使用。 Redis 支持多种数据结构，包括字符串、哈希表、链表、集合、有序集合、位图等。 Redis 具备 LRU 淘汰、事务实现、以及不同级别的硬盘持久化等能力，并且支持副本集和通过 Redis Sentinel 实现的高可用方案，同时还支持通过 Redis Cluster 实现的数据自动分片能力。Redis 支持的数据类型包括：字符串、哈希表、链表、集合、有序集合以及基于这些数据类型的相关操作。Redis 使用 C 语言开发，在大多数 `*nix` 系统上无需任何外部依赖就可以使用。Redis 支持的客户端语言也非常丰富，常用的计算机语言如 C 、 C# 、 C++ 、 Object-C 、 PHP 、Python、Java 、 Perl 、 Lua 、 Erlang 等均有可用的客户端来访问 Redis 服务器。当前 Redis 的应用已经非常广泛，国内像新浪、淘宝，国外的 Github 等均在使用 Redis 的缓存服务。

## Redis 的安装配置

我们之前提到过容器的一大优点就是安装配置极为容易，和网上的一些教程不同，我们这里不讲如何原生安装 redis ，我们直接就使用容器搞起。

第一步，取得 redis 的镜像， `redis-alphine` 是一个最小化的镜像版本。

```bash
docker pulll redis-alpine
```

你也可以不使用 `alpine` 版本，那就使用 `redis` ，那个 `:` 后面跟的是 `tag` ，一般可以是版本号，具体可以去 <https://hub.docker.com/_/redis/> 查看。

```bash
docker pulll redis:4.2.1
```

第二步，在 `docker-compose.yml` 中配置，

```yml
version: '3.2'
services:
  redis:
    # 使用那个 tag 的 image
    image: redis:4-alpine
    # 如果希望启动 redis 时带有参数，那么可以在此处定义
    command: [ "redis-server", "--protected-mode", "no" ]
    ports:
      - "6379:6379"
   volumes:
     # 映射文件夹
     - redis-data:/data
    hostname: redis
  mongo:
    image: mongo:3.6.4
    ports:
      - "27017:27017"
    volumes:
      - api_db:/data/db
  api-server:
    image: api
    ports:
      - "8080:8080"
      - "5005:5005"
    links:
      - mongo
      # 连接到 api-server，分开写成 redis:redis
      # 前面的是在这个 services 中的名字，后一个是在 api-service 中可见的 host 名称。
      - "redis:redis"
volumes:
  api_db: {}
  redis-data: {}
```

第三步，启动服务，我们只需在 `docker-compost.yml` 所在目录下执行下面的命令，就可以启动 redis 了，是不是很简单，到此服务就启动了，我们可以节省很多时间专注在开发上。

```bash
docker-compose up -d redis
```

有的同学说了，那我如果想同时启动 redis 和 mongo 咋办呢？敲下面这个命令。

```bash
docker-compose up -d redis mongo
```

对的，你猜到了，如果要连 api-service 一起启动，那就是

```bash
docker-compose up -d
```

那么如果要停止服务呢？下面的一句就解决问题了。

```bash
docker-compose down
```

那么到这里 Redis 的安装和配置就 ok 了，我们下面看看如何在 Spring Boot 中集成。

## 在 Spring Boot 中集成 Redis

Spring Boot 对于 Redis 有着开箱即用的体验，只需添加 `spring-boot-starter-data-redis` 即可完成 redis 的集成。

```groovy
// 省略
dependencies {
    // 省略
    implementation("org.springframework.boot:spring-boot-starter-data-mongodb")
    implementation("org.springframework.boot:spring-boot-starter-data-redis") // <--- 这里
    testImplementation("org.springframework.security:spring-security-test")
}
```

使用 Redis 作为缓存框架更是简单得不能再简单，只需要在 `application.yml` 中设置 `spring.cache.type` 为 `redis` ，随后就可以在 `spring.redis.host` 中配置主机名或 IP 地址，用 `spring.redis.port` 配置端口。在下面的配置中，我们为开发环境 ( `dev` ) 配置了 `localhost` ，而为生产环境配置了 `redis` 作为主机名，我们在生产环境中会使用容器，你还记得我们在 `docler-compose.yml` 中是怎么样配置的吧，我们的 `api-service` 连接了两个容器，一个是 `mongo` ，另一个就是 `redis` 。

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
    type: redis # <--- 默认使用 redis 作为缓存
  redis:
    host: redis # <--- redis 主机名，需要和 docker-compose.yml 中的 `link` 中定义保持一致
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
    host: localhost # <--- 开发模式下
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
    type: none  # 测试模式下，不使用 Cache
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

现在如果我们启动 MongoDB 和 Redis

```bash
docker-compose up -d redis mongo
```

然后在启动自己的应用，你就可以看到我们的缓存还是正常工作的，像之前一样。

### 在容器中配置 Redis

配置容器时，只需要在 `ENTRYPOINT` 配置启动的 `java` 命令时传入 `-Dspring.redis.host=redis` 即可，这里的 `host` 设置成 `redis` ，是因为 `docker-compose` 配置时，我们为 `api-server` 连接的就是 `redis` 。当然如果端口不是默认的 `6379` 的话，还需要传入 `-Dspring.redis.port=xxxx`

```groovy
// 省略
task createDockerfile(type: com.bmuschko.gradle.docker.tasks.image.Dockerfile, dependsOn: ['bootJar']) {
    description = "自动创建 Dockerfile"
    destFile = project.file('src/main/docker/Dockerfile')
    from 'openjdk:8-jdk-alpine'
    volume '/tmp'
    addFile "${project.name}-${project.version}.jar", "app.jar"
    instruction { 'ENTRYPOINT [' +
            '"java", ' +
            '"-agentlib:jdwp=transport=dt_socket,address=5005,server=y,suspend=n", ' +
            '"-Dspring.data.mongodb.uri=mongodb://mongo/taskmgr", ' +
            '"-Dspring.redis.host=redis", ' + // <--- 这里
            '"-Dspring.profiles.active=prod", ' +
            '"-Djava.security.egd=file:/dev/./urandom", ' +
            '"-jar","/app.jar"]'}
    maintainer 'Peng Wang "wpcfan@gmail.com"'
}
// 省略
```

## Redisson

Redisson 是一个在 Redis 的基础上实现的 Java 驻内存数据网格（ In-Memory Data Grid ）。它底层采用的是 Netty 框架，不仅提供了一系列的分布式的 Java 常用对象，还提供了许多分布式服务。其中包括( BitSet, Set, Multimap, SortedSet, Map, List, Queue, BlockingQueue, Deque, BlockingDeque, Semaphore, Lock, AtomicLong, CountDownLatch, Publish / Subscribe, Bloom filter, Remote service, Spring cache, Executor service, Live Object service, Scheduler service ) Redisson 提供了使用 Redis 的最简单和最便捷的方法。

提供一个 Redisson 的配置，我们需要在 `config` 包下新建 `CacheConfig.java`

```java
@RequiredArgsConstructor
@EnableCaching
@Configuration
@AutoConfigureAfter(RedisAutoConfiguration.class)
public class CacheConfig {

    private final RedisProperties redisProperties;

    @Bean(destroyMethod = "shutdown")
    RedissonClient redissonClient() {
        Config config = new Config();
        // sentinel
        if (redisProperties.getSentinel() != null) {
            SentinelServersConfig sentinelServersConfig = config.useSentinelServers();
            sentinelServersConfig.setMasterName(redisProperties.getSentinel().getMaster());
            val nodes = redisProperties.getSentinel().getNodes();
            sentinelServersConfig.addSentinelAddress(nodes.toArray(new String[0]));
            sentinelServersConfig.setDatabase(redisProperties.getDatabase());
            if (redisProperties.getPassword() != null) {
                sentinelServersConfig.setPassword(redisProperties.getPassword());
            }
        } else { // 单个 Server
            SingleServerConfig singleServerConfig = config.useSingleServer();
            String schema = redisProperties.isSsl() ? "rediss://" : "redis://";
            singleServerConfig.setAddress(schema + redisProperties.getHost() + ":" + redisProperties.getPort());
            singleServerConfig.setDatabase(redisProperties.getDatabase());
            if (redisProperties.getPassword() != null) {
                singleServerConfig.setPassword(redisProperties.getPassword());
            }
        }
        return Redisson.create(config);
    }
}
```

为了避免多处配置同样的信息，我们在这个文件中注入了 Redis 的属性设置类 `RedisProperties` 。根据是否设置了前哨而决定是构造一个分布式的配置还是单机配置。在单机配置中，我们利用 `redisProperties` 取得 redis 的信息，并应用到 Redisson 的配置中。在单机配置中， Redisson 设置的地址的格式是 `协议://地址:端口号` ，和 MongoDB 类似的， <redis://localhost:6379> 就是采用非 `SSL` 协议连接本机的 `6379` 端口，而 <rediss://redis:6479> 就是以 `SSL` 协议连接主机 `redis` 的 `6479` 端口。
