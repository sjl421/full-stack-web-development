# Spring Boot Actuator 和数据审计

在企业级开发中，我们经常会碰到需要进行数据审计，以及保留数据变更历史的需求。一般来说，当我们听到审计这个字眼儿的时候，浮现在脑海的应该是构建一个包含每个需要审核的实体的每个版本的日志。这个任务其实比较复杂，但所幸的是，现在我们不需要从头做起了。

Spring Boot Actuator 提供了一个监控和管理生产环境的模块，可以使用 `http` 、 `jmx` 、 `ssh` 、 `telnet` 等管理和监控应用。审计（ Auditing ）、 应用健康（ Health ）、数据采集（ Metrics Gathering ）会自动加入到应用里面。其中审计功能提供了对于安全事件的发布和订阅，默认的事件是鉴权成功或失败，但是可以自定义事件。这就给了我们一个机制可以处理企业需求中常见的审计功能。

## 初窥审计事件

如果我们向 `build.gradle` 中加入 Spring Boot Actuator 的依赖的话，Spring Boot 就会聪明的、自动的帮我们添加一系列的监控接口。

```groovy
dependencies {
  // 省略
  implementation("org.springframework.boot:spring-boot-starter-actuator")
  // 省略
}
```

### 监控管理接口

Spring Boot Actuator 内建了一系列的管理接口，这些接口的默认路径前缀是 `/actuator` ，比如说 `info` 的完整路径就是 <http://localhost:8080/actuator/info> 。

| 接口名称      | 描述                                                 | 默认可用 | JMX | Web |
| -------------- | ---------------------------------------------------- | --------- | --- | --- |
| auditevents    | 得到当前应用的审计事件                              | 是        | 是  | 否  |
| beans          | 列出当前应用的所有 Bean                              | 是        | 是  | 否  |
| conditions     | 列出用于匹配配置中的类的条件，以及它们是否匹配的原因 | 是        | 是  | 否  |
| configprops    | 列出所以定义为 @ConfigurationProperties 的属性       | 是        | 是  | 否  |
| env            | 列出 Spring 中的环境变量                             | 是        | 是  | 否  |
| flyway*        | 列出 flyway 中已经应用的 migration                  | 是        | 是  | 否  |
| health         | 应用的健康状况                                       | 是        | 是  | 是  |
| httptrace      | 显示 HTTP 跟踪信息，默认最近的 100 条请求响应信息  | 是        | 是  | 否  |
| info           | 显示应用信息                                         | 是        | 是  | 是  |
| loggers        | 获取或更改应用的日志信息                             | 是        | 是  | 否  |
| liquibase**    | 列出 Liquibase 已经应用的 migration                  | 是        | 是  | 否  |
| metrics        | 列出当前应用的指标信息                              | 是        | 是  | 否  |
| mappings       | 列出所有 @RequestMapping 路径                        | 是        | 是  | 否  |
| scheduledtasks | 列出应用中的定时任务                                 | 是        | 是  | 否  |
| sessions       | 允许获得和删除用户会话，不适用于 Reactive 模式      | 是        | 是  | 否  |
| shutdown       | 关闭应用                                             | 否        | 是  | 否  |
| threaddump     | 执行一个线程的 dump                                  | 是        | 是  | 否  |

```txt
* flyway 是一个开源的 SQL 数据库版本管理项目，项目地址在： <https://flywaydb.org/> ，该接口只有在集成了该类库之后才能生效。
** liquibase 是一个开源的 SQL 数据库版本管理项目，项目地址在： <https://www.liquibase.org/> ，该接口只有在集成了该类库之后才能生效。
```

### 监控接口的配置

从上面的表格可以看到，大部分接口默认都是 JMX 可访问，默认 Web 可用的只有 `health` 和 `info` 接口，而且默认情况下也禁止了跨域 CORS 。但能否设置成 Web 可访问呢？当然可以了，只需要在 `application.yml` 中做一些设置即可：

```yml
management:
  endpoints:
    # 默认激活所有监控管理接口
    enabled-by-default: true
    web:
      # 设置路径前缀，默认是 /actuator
      base-path: /management
      # 让所有监控接口都接受 web 访问
      exposure:
        include: "*"
      # 设置跨域
      cors:
        allowed-origins: "*"
        allowed-methods: GET,POST
```

上面我们通过设置 `base-path` 也把路径前缀改成了 `/management` ，现在我们启动应用，然后访问 <http://localhost:8080/management> ，这一次，我们应该得到了更多的接口信息：

![开放全部管理接口](/assets/2018-05-20-11-44-14.png)

如果此时我们启动应用，然后调用 `/auth/login` 接口，做一次鉴权，成功还是失败是无所谓的，之后我们访问 <http://localhost:8080/management/auditevents> 就可以看到鉴权事件了。

![返回审计事件](/assets/2018-05-20-12-25-56.png)

我们把这个返回的 `json` 格式化一下，这样看的更清楚一些，我们得到是一个 `events` 数组，数组中的每个元素都是一个 `org.springframework.boot.actuate.audit.AuditEvent` ：

```json
{
  "events":[
    {
      "timestamp":"2018-05-20T04:23:46.796Z",
      "principal":"wpcfan",
      "type":"AUTHENTICATION_FAILURE",
      "data":{
        "type":"org.springframework.security.authentication.BadCredentialsException",
        "message":"Bad credentials"
      }
    }
  ]
}
```

### 监控接口的安全

大部分的监控接口我们是不想暴露给普通用户的，否则也太危险了，想想看如果 `shutdown` 接口被远程的非法用户调用的画面。所以我们需要对接口进行安全保护，所幸的是，我们由于已经集成了 Spring Security ，这一点要做到非常容易，只需要在 `SecurityConfig` 中来设置对应接口的权限。

```java
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    // 省略

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.addFilterBefore(corsFilter, UsernamePasswordAuthenticationFilter.class)
          .exceptionHandling()
          .authenticationEntryPoint(problemSupport)
          .accessDeniedHandler(problemSupport)
          .and()
            .csrf().disable()
            .headers().frameOptions().disable()
          .and()
            .sessionManagement()
            .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
          .and()
            .authorizeRequests()
            .antMatchers("/api/auth/**").permitAll()
            .antMatchers("/api/**").authenticated()
            // 允许健康接口匿名访问
            .antMatchers("/management/health").permitAll()
            // 其他监控接口都需要管理员权限
            .antMatchers("/management/**").hasAuthority(AuthoritiesConstants.ADMIN)
            .antMatchers("/v2/api-docs/**").permitAll()
            .antMatchers("/swagger-resources/configuration/ui").permitAll()
            .antMatchers("/swagger-ui/index.html").permitAll()
          .and()
            .apply(jwtConfigurer);
    }
}
```

## 实现应用的数据审计

通常情况下，审计的需求归结为两个问题：

1. 实体的更改或创建发生在什么时间？
2. 谁执行的这次变更？

既然这两个问题属于审计的通用问题，那么所有要求审计的实体就得能够回答这两个问题。因此我们构造一个抽象基类 `AbstractAuditingEntity` ，所有在本应用中需要审计的实体均要继承这个基类。

```java
package dev.local.gtm.api.domain;

import com.fasterxml.jackson.annotation.JsonIgnore;
import lombok.Getter;
import lombok.Setter;
import org.springframework.data.annotation.CreatedBy;
import org.springframework.data.annotation.CreatedDate;
import org.springframework.data.annotation.LastModifiedBy;
import org.springframework.data.annotation.LastModifiedDate;
import org.springframework.data.mongodb.core.mapping.Field;

import java.io.Serializable;
import java.time.Instant;


/**
 * 审核特性的领域对象的基础抽象类，包括以下属性：创建人，创建时间，修改人，修改时间
 * 一般情况下如果你希望领域对象有历史操作记录，需要让领域对象继承此基类。
 *
 * @author Peng Wang (wpcfan@gmail.com)
 */
@Getter
@Setter
public abstract class AbstractAuditingEntity implements Serializable {

    private static final long serialVersionUID = 1L;

    @CreatedBy
    @Field("created_by")
    @JsonIgnore
    private String createdBy;

    @CreatedDate
    @Field("created_date")
    @JsonIgnore
    private Instant createdDate = Instant.now();

    @LastModifiedBy
    @Field("last_modified_by")
    @JsonIgnore
    private String lastModifiedBy;

    @LastModifiedDate
    @Field("last_modified_date")
    @JsonIgnore
    private Instant lastModifiedDate = Instant.now();
}
```

我们在这个基类中定义了创建者（ `createdBy` ）、创建时间（ `createdDate` ）、最近修订者（ `lastModifiedBy` ）以及最近修改时间（ `lastModifiedDate` ）。而且我们使用了 Spring Data 提供的针对这几个特性的专门注解：`@CreatedBy` 和 `@LastModifiedBy` 用于获得谁创建或修改了实体，而 `@CreatedDate` 和 `@LastModifiedDate` 用于获取发生的时间点。

当我们使用 `@CreatedBy` 或 `@LastModifiedBy` 注解之后，审计框架需要知道当前操作的用户信息，所以我们要提供一个 `AuditorAware<T>` 的实现类，让审计框架可以通过 `getCurrentAuditor` 方法得到当前操作用户。

```java
package dev.local.gtm.api.security;

import dev.local.gtm.api.config.Constants;
import org.springframework.data.domain.AuditorAware;
import org.springframework.stereotype.Component;

import java.util.Optional;

@Component
public class SpringSecurityAuditorAware implements AuditorAware<String> {

    @Override
    public Optional<String> getCurrentAuditor() {
        return Optional.of(SecurityUtils.getCurrentUserLogin().orElse(Constants.SYSTEM_ACCOUNT));
    }
}
```

当然如果要实现比较简单的审计功能，这样就可以了，在实体保存到数据库时，Spring Boot 会自动设置创建/最近修改时间以及创建人和修改人。当然，我们为了让 Spring Boot 激活审计功能，还需要在 `DatabaseConfig` 中增加一个注解 `@EnableMongoAuditing`

```java
@EnableMongoAuditing(auditorAwareRef = "springSecurityAuditorAware")
public class DatabaseConfig {
  // 省略
}
```

但是如果是比较复杂的审计功能，比如要进行对象的版本比较等的话，自己要实现的话就比较复杂了。好在有人已经为这种需求提供了强大的解决方案，而且和 Spring Boot 的配合及其良好。下面，我们就有请 `JaVers` 闪亮登场。

## JaVers 和 Spring Boot 集成

Javer <https://javers.org/> 是一个开源的专注于数据审计和版本控制的软件，它可以和 Spring Boot ，Spring Security 进行无缝集成，提供 SQL 数据库和 MongoDB 的良好支持。

集成 JaVers 的步骤非常简单，首先在 `build.gradle` 中添加依赖

```groovy
implementation("org.javers:javers-spring-boot-starter-mongo:3.9.7")
```

然后新建一个 `AuthorProvider` 的实现类，用来提供当前操作用户。

```java
package dev.local.gtm.api.config.audit;

import org.javers.spring.auditable.AuthorProvider;
import org.springframework.stereotype.Component;

import dev.local.gtm.api.config.Constants;
import dev.local.gtm.api.security.SecurityUtils;

@Component
public class JaversAuthorProvider implements AuthorProvider {

  @Override
  public String provide() {
    return SecurityUtils.getCurrentUserLogin().orElse(Constants.SYSTEM_ACCOUNT);
  }
}
```

最后，对于要进行数据审计的 Repository ，添加一个注解，拿 `TaskRepository` 来说，我们添加 `@JaversSpringDataAuditable` 注解之后， JaVers 就会自动处理 `Task` 的变更。

```java
package dev.local.gtm.api.repository.mongo;

// 省略其他导入
import org.javers.spring.annotation.JaversSpringDataAuditable;

/**
 * 任务存储接口
 */
@JaversSpringDataAuditable
@Repository
public interface TaskRepository extends MongoRepository<Task, String> {
    // 省略
}

```

JaVers 会在 MongoDB 中创建一个叫 `jv_snapshots` 的 collection ，所有的数据变化都会存储在这个 collection 中。

我们现在需要建立一个 Service ，用于获得审计对象的变化历史列表以及之前一个版本的变更。需要注意的是我们需要一个带完整包名的类名称，以便得到待审计对象的类型（ `Class.forName` ），然后通过 JaVers 提供的 `QueryBuilder` 构造对应的查询。

```java
package dev.local.gtm.api.service;

import org.javers.core.Javers;
import org.javers.repository.jql.QueryBuilder;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageImpl;
import org.springframework.data.domain.Pageable;
import org.springframework.security.access.annotation.Secured;
import org.springframework.stereotype.Service;

import dev.local.gtm.api.domain.EntityAuditEvent;
import dev.local.gtm.api.security.AuthoritiesConstants;

import java.util.Optional;
import java.util.stream.Collectors;

import lombok.RequiredArgsConstructor;
import lombok.val;
import lombok.extern.slf4j.Slf4j;

@Slf4j
@RequiredArgsConstructor
@Service
public class AuditEventService {

  private static final String basePackageName = "dev.local.gtm.api.domain.";
  private final Javers javers;

  @Secured(AuthoritiesConstants.ADMIN)
  public Page<EntityAuditEvent> getChanges(String entityType, Pageable pageable) throws ClassNotFoundException {
    log.debug("获得一页指定实体对象类型的审计事件");
    val entityTypeToFetch = Class.forName(basePackageName + entityType);
    val jqlQuery = QueryBuilder.byClass(entityTypeToFetch)
      .limit(pageable.getPageSize())
      .skip(pageable.getPageNumber() * pageable.getPageSize())
      .withNewObjectChanges(true);

    val auditEvents = javers.findSnapshots(jqlQuery.build()).stream()
      .map(snapshot -> {
        EntityAuditEvent event = EntityAuditEvent.fromJaversSnapshot(snapshot);
        event.setEntityType(entityType);
        return event;
      })
      .collect(Collectors.toList());

    return new PageImpl<EntityAuditEvent>(auditEvents);
  }

  @Secured(AuthoritiesConstants.ADMIN)
  public EntityAuditEvent getPrevVersion(String entityType, String entityId, Long commitVersion)
      throws ClassNotFoundException {
    val entityTypeToFetch = Class.forName(basePackageName + entityType);

    val jqlQuery = QueryBuilder.byInstanceId(entityId, entityTypeToFetch)
      .limit(1)
      .withVersion(commitVersion - 1)
      .withNewObjectChanges(true);

    return EntityAuditEvent.fromJaversSnapshot(javers.findSnapshots(jqlQuery.build())
      .get(0));
  }
}
```

有了 Service 之后，对应的 Controller 就非常简单了，简单的调用上面建好的 `AuditEventService` 即可。

```java
package dev.local.gtm.api.web.rest;

import org.elasticsearch.ResourceNotFoundException;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.security.access.annotation.Secured;
import org.springframework.web.bind.annotation.*;

import dev.local.gtm.api.domain.EntityAuditEvent;
import dev.local.gtm.api.security.AuthoritiesConstants;
import dev.local.gtm.api.service.AuditEventService;
import lombok.RequiredArgsConstructor;

import java.util.Arrays;
import java.util.List;

@RequiredArgsConstructor
@RestController
@RequestMapping("/api")
public class AuditResource {

  private final AuditEventService auditEventService;

  @GetMapping("/management/audits/entities")
  @Secured(AuthoritiesConstants.ADMIN)
  public List<String> getAuditedEntities() {
    return Arrays.asList("User", "Authority", "Task");
  }

  @GetMapping("/management/audits/changes")
  public Page<EntityAuditEvent> getChanges(@RequestParam("entityType") String entityType, Pageable pageable) {
    try {
      return auditEventService.getChanges(entityType, pageable);
    } catch(ClassNotFoundException e) {
      throw new ResourceNotFoundException("指定的类型没有找到");
    }
  }

  @GetMapping("/management/audits/previous")
  public EntityAuditEvent getPreviousVersion(
    @RequestParam String entityType,
    @RequestParam String entityId,
    @RequestParam Long commitVersion) {
    try {
      return auditEventService.getPrevVersion(entityType, entityId, commitVersion);
    } catch (ClassNotFoundException e) {
      throw new ResourceNotFoundException("指定的类型没有找到");
    }
  }
}
```

那么我们可以测试一下效果
