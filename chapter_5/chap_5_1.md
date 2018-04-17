# HyperMedia API vs 传统 API

## 领域对象

和前面前端类似，我们在后端也需要定义领域对象。而且从某种角度说，后端的领域对象往往也影响前端的建模。既然是登录鉴权，那么我们肯定要建立一个 `User` 对象。这个和前端的模型很类似，没什么好说的。

```java
public class User {
    private String username;
    private String password;
    private String mobile;
    private String name;
    private String email;
    private Gender gender;
    private String avatar;
}
```

上面这个类定义完成了吗？没有，在 Java 中有一个很烦的地方就是要写好多 `getter` 和 `setter` ，还有构造函数、 `hashCode` , `equals` 和 `toString` 等方法，当然其实现代 IDE 都提供了一些快捷方式生成。但是还是有些麻烦，最重要的是，这些方法在查找问题、分析代码和评审代码时会分散注意力。有没有什么方法可以让我们从这些模式化代码中解脱出来呢？有，那就是 Java 社区大名鼎鼎的 `lombok` ，可以访问其官网 <https://projectlombok.org> 了解更多特性的详情。

### 使用 Lombok 简化模式化代码的编写

#### 配置 Lombok

在 `api/build.gradle` 中，加入 `lombok` 的依赖， `lombok` 也提供了 Spring Boot 的支持，所以不用写版本号。

```groovy
apply plugin: 'org.springframework.boot'

dependencies {
    implementation("org.springframework.boot:spring-boot-devtools")
    implementation("org.projectlombok:lombok")
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-data-rest")
    implementation("org.springframework.boot:spring-boot-starter-data-mongodb")
    implementation("org.springframework.data:spring-data-rest-hal-browser")
}
```

然后在 IDEA 中的 `Preference > Build, Execution, Deployment > Complier > Annotation Processors` ，选中 `Enable annotation processors` 然后点击 OK 。

![在 IDEA 启用 Annotation Processors](/assets/2018-04-11-15-35-34.png)

这样就完成了 Lombok 的设置，下面我们就简单的了解一下 Lombok 可以给我带来哪些魔法。

#### 大杀器 @Data

Lombok 提供了很多注解，但其中最常被使用的就是这个 `@Data` 了 <https://projectlombok.org/features/Data> ，只需放一个注解在类前面，一切烦恼就都消失了，可谓居家旅行必备良药。这么一个注解会帮你生成所有属性的 `get` 和 `set` 方法 （ `final` 属性只生成 `get` 方法），帮你实现 `hashCode` , `equals` 和 `toString` 方法以及由必选参数（就是所有 `final` 属性以及 标识 `@NonNull` 注解的属性）构成的构造函数。

最棒的是，这些魔法是在编译时完成的，和 IDE 帮你自动生成的代码不同，你的源文件中永远都不会出现这些冗长的代码，一直就是这种短小精悍的形式，这对于开发者实在是太友好了。

```java
package dev.local.gtm.api.domain;

import dev.local.gtm.api.domain.enums.Gender;
import lombok.Data;

@Data // <----------这里
public class User {
    private String username;
    private String password;
    private String mobile;
    private String name;
    private String email;
    private Gender gender;
    private String avatar;
}
```

#### @Builder 让对象创建无烦恼

这个注解也是笔者挚爱之一，在介绍它之前，我们先来看一段代码。

```java
User newUser = new User();
newUser.setUsername(userDTO.getUsername());
newUser.setPassword(encryptedPassword);
newUser.setFirstName(userDTO.getFirstName());
newUser.setLastName(userDTO.getLastName());
newUser.setEmail(userDTO.getEmail());
newUser.setImageUrl(userDTO.getImageUrl());
newUser.setLangKey(userDTO.getLangKey());
newUser.setActivated(false);
```

是不是有种很熟悉的感觉，先 `new` 出来对象，然后一列齐刷刷的 `newUser.setXXX` 个人感觉是很干扰阅读的，而且在设置属性的时候，重新敲 `newUser.setXXX` 总有思路被打断的感觉。现代 API 写法中越来越多的使用了 Builder 模式，那么 Lombok 的注解 @Builder 又帮你省去了手写的工作量，可以优雅的写 Builder 模式的代码，何乐而不为呢？

```java
User.builder()
  .username(userDTO.getUsername())
  .password(encryptedPassword)
  .firstName(userDTO.getFirstName())
  .lastName(userDTO.getLastName())
  .email(userDTO.getEmail())
  .imageUrl(userDTO.getImageUrl())
  .langKey(userDTO.getLangKey())
  .activated(false)
  .build();
```

此外和 `@Builder` 经常在一起使用的有 `@Singular` ，这个注解标识类中的集合属性，为集合属性生成添加单个元素和集合元素以及清空集合的方法。详细情况可以参考官网的解释 <https://projectlombok.org/features/Builder> 。

#### 别烦我了 try...finally

Java 的异常处理当然是很强大的，但是当你遇到下面这种情况，是不是也会觉得生无可恋啊？

```java
import java.io.*;

public class CleanupExample {
  public static void main(String[] args) throws IOException {
    InputStream in = new FileInputStream(args[0]);
    try {
      OutputStream out = new FileOutputStream(args[1]);
      try {
        byte[] b = new byte[10000];
        while (true) {
          int r = in.read(b);
          if (r == -1) break;
          out.write(b, 0, r);
        }
      } finally {
        if (out != null) {
          out.close();
        }
      }
    } finally {
      if (in != null) {
        in.close();
      }
    }
  }
}
```

Lombok 可以让你这样写，世界顿时清净了吧，有木有？

```java
import lombok.Cleanup;
import java.io.*;

public class CleanupExample {
  public static void main(String[] args) throws IOException {
    @Cleanup InputStream in = new FileInputStream(args[0]);
    @Cleanup OutputStream out = new FileOutputStream(args[1]);
    byte[] b = new byte[10000];
    while (true) {
      int r = in.read(b);
      if (r == -1) break;
      out.write(b, 0, r);
    }
  }
}
```

#### 其他几个常用注解

除去上面的几个注解，还有一些我们在本书中会用到的一些 Lombok 提供的工具，它们没有那么强大的功能，但同样会给你带来生产效率的提升。

* `val` 和 `var` -- 用过 swift 或 kotlin 的读者肯定知道 `val` 和 `var` ，前者是不可变对象，后者是可变对象。有了 Lombok ，你就在 Java 中也可以这么使用了 `val example = new ArrayList<String>();` ，而不用写成 `final ArrayList<String> example = new ArrayList<String>();` 。 `var` 就不举例了，除了对象可变之外，其他和 `val` 一样。
* `@Log` 、 `@Slf4j` 、 `@Log4j`、 `@CommonsLog` 等等 -- 写日志是我们在编程中经常遇到的，Lombok 提供的这个注解在类上标注后，直接程序中写 `log.error()` 等走起。

### 使用 Lombok 改造领域对象

工具介绍完毕，我们回到领域对象中，使用 Lombok 来改造领域对象 `User`。

```java
import com.fasterxml.jackson.annotation.JsonIgnore;
import dev.local.gtm.api.config.Constants;
import lombok.*;
import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.index.Indexed;
import org.springframework.data.mongodb.core.mapping.Field;

import javax.validation.constraints.Email;
import javax.validation.constraints.NotNull;
import javax.validation.constraints.Pattern;
import javax.validation.constraints.Size;
import java.io.Serializable;
import java.time.Instant;

@Getter
@Setter
@EqualsAndHashCode(of = "id")
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class User implements Serializable {
    private static final long serialVersionUID = 1L;

    @Id
    private String id;

    @NotNull
    @Pattern(regexp = Constants.LOGIN_REGEX)
    @Size(min = 1, max = 50)
    @Indexed
    private String login;

    @JsonIgnore
    @NotNull
    @Size(min = 60, max = 60)
    private String password;

    @NotNull
    @Pattern(regexp = Constants.MOBILE_REGEX)
    @Size(min = 10, max = 15)
    private String mobile;

    @Size(max = 50)
    private String name;

    @Email
    @Size(min = 5, max = 254)
    @Indexed
    private String email;

    @Size(max = 256)
    private String avatar;

    @Builder.Default
    private boolean activated = false;
}
```

这样看起来一个领域对象的代码就清晰多了。比起之前的 `User` ，我们添加了一些字段的约束，比如 `@NotNull` 、 `@Pattern` 、 `@Size` 、 `@Email` 这些都属于 `JSR 380 Java Bean Validation` 提供的注解，用以确保属性满足指定条件。而且这些注解还可以和 Java 8 的 `Optional` 连用：

```java
private LocalDate dateOfBirth;

public Optional<@Past LocalDate> getDateOfBirth() {
    return Optional.of(dateOfBirth);
}
```

而 `@Indexed` 属于 Spring Data MongoDB 提供的注解，这个注解顾名思义就是建立索引，查看索引的话，可以登录到 MongoDB 的容器中，使用 MongoDB 的客户端命令进行操作。当然这个索引目前还不是程序规定的样子，直到有数据写入时索引会自动创建。

```bash
## 先登录到容器
docker exec -it [容器名称] bash
## 然后进入 MongoDB 的客户端
mongo
## 输入下面的命令查看索引
db.user.getIndexes()
```

对于使用的 Lombok 注解，需要指出 `@Builder.Default` 这个注解是如果在有属性的默认值时，同时又使用了 `@Builder` 的情况下，用来标记有默认值的那个属性。再有就是我们没有使用 `@Data` 的原因是，对于用户的 `equals` 和 `hashCode` 两个方法，我们不想比较所有字段，其实只需比较 `id` 即可，而 `@Data` 默认是全部比较的。指定哪些属性参与到 `equals` 和 `hashCode` 方法中的比较，可以使用 `@EqualsAndHashCode(of = {"属性1", "属性2", ...})` 来去实现。

## API 的可见控制

有细心的同学可能发现了，之前我们的 API 默认暴露了 `repository` 中支持的全部接口。如果启动服务，使用浏览器访问 `http://localhost:8080` 进入 `HAL Browser` ，在 `Explorer` 的下方输入框内输入 `/profile/users` 点击 `Go!` 可以看到 Response Body 中返回的 `json` 。

```json
{
  "alps": {
    "version": "1.0",
    "descriptors": [
      // 省略其他部分
      {
        "id": "create-users",
        "name": "users",
        "type": "UNSAFE",
        "rt": "#user-representation"
      },
      {
        "id": "get-user",
        "name": "user",
        "type": "SAFE",
        "rt": "#user-representation"
      },
      {
        "id": "update-user",
        "name": "user",
        "type": "IDEMPOTENT",
        "rt": "#user-representation"
      },
      {
        "id": "patch-user",
        "name": "user",
        "type": "UNSAFE",
        "rt": "#user-representation"
      },
      {
        "id": "delete-user",
        "name": "user",
        "type": "IDEMPOTENT",
        "rt": "#user-representation"
      }
    ]
  }
}
```

从上面的 json 中，可以看到我们暴露了增、删、改、查等操作，大家可以通过 Postman 进行验证，所有的操作都是允许的。 那么问题来了，很多时候我们只想开放某些接口，比如只开放只读接口，不允许写操作。这种情况下怎么办呢？答案非常简单，还是采用注解，但是需要配合适当的策略。

Spring Data Rest 使用 `RepositoryDetectionStrategy` 来决定一个 `repository` 是否作为 Rest 资源暴露出去。需要指出的是 `RepositoryDetectionStrategy` 只决定 API 的外部可见策略，并不涉及权限，也就是说能看到不代表能执行。

* `DEFAULT` -- 默认情况下会把所有的 `public` 的 `respository` 接口暴露成 Rest 资源。但是仍然会尊重 `@RepositoryRestResource` 或 `@RestResource` 的 `exported` 标志位。也就是说，比如某个 `TaskRepo` ，我们加上这个注解 `@RepositoryRestResource(collectionResourceRel = "tasks", path = "tasks", exported = false)` ，那么我们在访问 `http://localhost:8080/api` 时将不会把 `api/tasks` 列出。
* `ALL` -- 暴露所有 `repository` 的接口方法
* `ANNOTATION` -- 只有添加了注解 `@RepositoryRestResource` 和 `@RestResource` 的 `repository` 方法会暴露。但仍会尊重 `exported` 标志位，也就是说如果 `exported = false` ，那么就不暴露该资源。
* `VISIBILITY` -- 只暴露 `public` 的添加了注解 `@RepositoryRestResource` 的 `repository` 接口。

这个策略的配置，有几个方法可以选择，其一是建立一个配置类

```java
package dev.local.gtm.api.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.data.rest.core.config.RepositoryRestConfiguration;
import org.springframework.data.rest.core.mapping.RepositoryDetectionStrategy;
import org.springframework.data.rest.webmvc.config.RepositoryRestConfigurerAdapter;

@Configuration
public class SpringRestConfiguration extends RepositoryRestConfigurerAdapter {
    @Override
    public void configureRepositoryRestConfiguration(RepositoryRestConfiguration config) {
        config.setRepositoryDetectionStrategy(RepositoryDetectionStrategy.RepositoryDetectionStrategies.DEFAULT);
    }
}
```

另一种推荐做法是通过配置文件，比如 `application.yml` 或 `application.properties` 来实现，如果没有复杂的自定义配置，推荐这种方式，因为更简单。

```yml
# application.yml
spring:
  # 省略其他部分
  data:
    rest:
      detection-strategy: default
```

### DEFAULT 默认策略

我们先来看一下 `default` 策略，首先把 `UserRepo` 改造成下面的样子，我们显式的给增、删、改等操作加上了注解 `@RestResource(exported = false)` ，就是说不想暴露成 Rest 资源。

```java
@Repository
public interface UserRepo extends MongoRepository<User, String> {


    @RestResource(exported = false)
    @Override
    <S extends User> S insert(S entity);

    @Override
    Page<User> findAll(Pageable pageable);

    @RestResource(exported = false)
    @Override
    <S extends User> S save(S entity);

    @Override
    Optional<User> findById(String s);

    @Override
    boolean existsById(String s);

    @Override
    long count();

    @RestResource(exported = false)
    @Override
    void deleteById(String s);
}
```

现在我们再去 `HAL Browser` 中，输入 `/profile/users` ，看看返回的 Response Body ，我们会发现允许的操作仅剩下了读操作。

```json
{
  "alps": {
    "version": "1.0",
    "descriptors": [
      // 省略其他部分
      {
        "id": "get-user",
        "name": "user",
        "type": "SAFE",
        "rt": "#user-representation"
      }
    ]
  }
}
```

其实在这种策略下，如果我们还是继承了 `MongoRepository` ，那么我们只标记不想暴露的操作即可，也就是说我们根本不用去覆写读操作的方法。

```java
@Repository
public interface UserRepo extends MongoRepository<User, String> {

    @RestResource(exported = false)
    @Override
    <S extends User> S insert(S entity);

    @RestResource(exported = false)
    @Override
    <S extends User> S save(S entity);

    @RestResource(exported = false)
    @Override
    void deleteById(String s);
}
```

上面的代码中我们只保留了对于写操作的三个方法，其效果和上面的是等价的，这是因为 `default` 策略默认开放所有 `public` 接口，我们的接口是继承了 `MongoRepository` ，而 `MongoRepository` 又继承了 `CrudRepository` 、 `PagingAndSortingRepository` 和 `Repository` ，这些接口中的 `public` 方法都会默认作为 Rest 资源，当然很多方法其实对应的是一个资源，比如 n 多的 findAllXXX 对应的都是 `users` 这个资源，只不过传递的参数不一样。在这种情况下，我们显式标注哪些不想暴露出去就可以了。

上面得到方式适合我们需要 `MongoRepository` 的很多方法，但对外不想暴露成 API ，内部的 Service 或 Controller 中还是需要这些未暴露的方法的。如果我们不需要哪些未暴露的方法，做法其实是从继承 `MongoRepository` 变为继承一个更上层的 `Repository` 类型，比如 `CrudRepository` 、 `PagingAndSortingRepository` 甚至 `Repository` 。在下面的例子中我们直接使用了 `Repository` 这个接口，注意这个接口和我们的注解重名，所以注解需要使用全名。这样我们就只暴露这三个接口，其他的方法无论是内部还是外部都没有提供。

```java
import dev.local.gtm.api.domain.User;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.repository.Repository;

import java.util.Optional;

@org.springframework.stereotype.Repository
public interface UserRepo extends Repository<User, String> {
    Page<User> findAll(Pageable pageable);

    Optional<User> findById(String s);

    long count();
}
```

### ALL 策略

```yml
# application.yml
spring:
  # 省略其他部分
  data:
    rest:
      detection-strategy: all
```

首先调整策略为 `all` , 说起来 `all` 和 `default` 区别可能很多同学搞不清楚，因为几乎是一样的，只不过对于 `public` 的界定不同。如下代码中我们的 `UserRepo` 和 `default` 策略那节的代码只相差一个 `public` 的范围限定，但下面的这个代码在 `all` 策略中就可以把接口暴露出来，但如果改成 `default` 策略的话，那么这个 `/users` 的资源是不存在的。

```java
@Repository
interface UserRepo extends MongoRepository<User, String> {

    @RestResource(exported = false)
    @Override
    <S extends User> S insert(S entity);

    @RestResource(exported = false)
    @Override
    <S extends User> S save(S entity);

    @RestResource(exported = false)
    @Override
    void deleteById(String s);
}
```

### ANNOTATION 策略

```yml
# application.yml
spring:
  # 省略其他部分
  data:
    rest:
      detection-strategy: annotated
```

如果采用 `ANNOTATION` 策略，所有要暴露的 `repository` 和 `repository` 方法都需要显式添加注解，不想暴露的方法也需要显式添加注解并设置 `exported = false` 。

```java
@RepositoryRestResource
public interface UserRepo extends MongoRepository<User, String> {

    @RestResource(exported = false)
    @Override
    <S extends User> S insert(S entity);

    @RestResource
    @Override
    Page<User> findAll(Pageable pageable);

    @RestResource(exported = false)
    @Override
    <S extends User> S save(S entity);

    @RestResource
    @Override
    Optional<User> findById(String s);

    @RestResource(exported = false)
    @Override
    void deleteById(String s);
}
```

### VISIBILITY 策略

```yml
# application.yml
spring:
  # 省略其他部分
  data:
    rest:
      detection-strategy: visibility
```

这种策略只暴露 `public` 的添加了注解 `@RepositoryRestResource` 的 `repository` 接口。也就是只有类似下面的接口才会暴露为 Rest 资源，缺少 `@RepositoryRestResource` 或者不是 `public` 的接口都不能暴露为 Rest 资源。

```java
@RepositoryRestResource
public interface UserRepo extends MongoRepository<User, String> {
  // 省略
}
```

`Spring Data Rest` 在构造 API 方面非常容易，但在目前的使用上还没有普及，大部分项目使用的还是 Level 2 的 API 。而且一些周边的开源类库和它的配合还不是太好，比如笔者成书的时间上 Spring 的 Swagger 集成类库 `SpringFox` 对于 `Spring Data Rest 3.x` 还是没有支持，个人感觉在目前阶段还不适合在生产项目中使用，但可以持续关注。所以接下来的项目实践中，我们还是改回成经典的 Rest 实现模式。

## 传统的 API 实现模式

首先我们可以先去掉 `api/build.gradle` 中的 Data Rest 依赖

```groovy
# 删除下面这个依赖
implementation("org.springframework.boot:spring-boot-starter-data-rest")
```

改造 `application.yml` 为

```yml
spring:
  application:
    name: api-service
  devtools:
    remote:
      secret: thisismysecret
  data:
    mongodb:
      database: gtm-api
```

### 找回熟悉的 Controller

对于很多习惯了 Spring 开发的同学来讲，`Controller` ，`Service` ，`DAO` 这些套路突然间都没了会有不适感。其实呢，这些东西还在，只不过对于较简单的情景下，这些都变成了系统背后帮你做的事情。这一小节我们就先来看看如何将  `Controller` 再召唤回来。

由于 `Controller` 其实就是定义 API 资源，所以我们在 rest 包下建立 `/api/web/rest/AuthResource.java` 。这样的语义比较清晰，既然是 rest 的资源就叫 `XXXResource` 。

如果要让 `AuthResource` 可以和 `UserRepo` 配合工作的话，我们当然需要在 `AuthResource` 中需要引用 `TaskRepo` 。

```java
@Log4j2
@RestController
@RequestMapping("/api")
@RequiredArgsConstructor
public class AuthResource {
    private final UserRepo userRepo;
    //省略其它部分
}
```

Spring 现在鼓励用构造函数来做注入，所以，我们使用 Lombok 提供的注解 `@RequiredArgsConstructor` 来自动生成一个构造函数，为什么叫做 `RequiredArgs` ？因为这个构造是使用必须的参数构成的，它相当于下面的代码，一般来说，我们如果将成员变量声明成 `private final blabla`， 那么这个 `blabla` 就是必须的参数，它必须通过构造函数赋值：

```java
@Log4j2
@RestController
@RequestMapping("/api")
@RequiredArgsConstructor
public class AuthResource {

    private final UserRepo userRepo;

    @AutoWired
    public AuthResource(UserRepo userRepo){
        this.userRepo = userRepo;
    }
    //省略其它部分
}
```

当然我们为了可以让 Spring 知道这是一个支持 REST API 的 Controller ，还是需要标记其为 `@RestController`。对 Spring 熟悉的同学可能知道还有另一个注解是 `@Controller` ，那么这个 `@RestController` 和 `Controller` 有什么区别呢？ `@RestController` 是 Spring 4.x 引入的一个注解，它相当于 `@Controller` + `@RepsonseBody` ，也就是说使用 `@RestController` 之后你不用给这个 Controller 的方法加上 `@RepsonseBody` 了，详情可以参考 <https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/bind/annotation/RestController.html> 。

拿第三章的例子来看，我们在 `getAllTasks()` 方法的声明里直接返回了 `List<Task>` ，如果使用 Postman 请求的话，会发现 Resposne Body 中有了这个数组列表。这个处理其实是通过 `@ResponseBody` 完成的。 如果使用 `@Controller` ，我们就得声明 `public @RepsonseBody List<Task> getAllTasks()` ，而 `@RestController` 就是进一步简化了我们的工作，只需在类上使用这个注解，然后在类的的方法上就不用使用 `@ResponseBody` 了。

```java
@RestController
public class TaskController {
    // 使用@RequstMapping指定可以访问的URL路径
    @RequestMapping("/tasks")
    public List<Task> getAllTasks() {
        // 省略
    }
}
```

接下来我们来实现我们的登录和注册 API ，在 `AuthResource` 中添加两个方法 `login` 和 `register` ：

```java
@Log4j2
@RestController
@RequestMapping("/api")
@RequiredArgsConstructor
public class AuthResource {

    private final UserRepo userRepo;

    @PostMapping(value = "/auth/login")
    public User login(@RequestBody final Auth auth) {
        log.debug("REST 请求 -- 将对用户: {} 执行登录鉴权", auth);
        val user = userRepo.findOneByLogin(auth.getLogin());
        if (!user.isPresent()) {
            throw new LoginNotFoundException();
        }
        if (!user.get().getPassword().equals(auth.getPassword())) {
            throw new InvalidPasswordException();
        }
        return user.get();
    }

    @PostMapping("/auth/register")
    public ResponseEntity<User> register(@RequestBody final User user) {
        log.debug("REST 请求 -- 注册用户: {} ", user);
        if (userRepo.findOneByLogin(user.getLogin()).isPresent()) {
            throw new LoginExistedException();
        }
        if (userRepo.findOneByMobile(user.getMobile()).isPresent()) {
            throw new MobileExistedException();
        }
        if (userRepo.findOneByEmail(user.getEmail()).isPresent()) {
            throw new MobileExistedException();
        }
        return ResponseEntity.ok(userRepo.save(user));
    }
}
```

上面的代码中需要再说明几个要点：

 1. Spring 4.3 之后引入了 `@PostMapping` 、 `@PutMapping` 、 `@DeleteMapping` 、 `@PatchMapping` 和 `@GetMapping` 这几个注解。这些注解和 `@RequestMapping` 的区别在于，它们进一步简化了注解的使用，因为已经各个注解已经隐含了 HTTP 的方法，比如 `@PostMapping` 就相当于 `@RequestMapping(method = RequestMethod.POST)` 。
 2. 这些方法接受的参数也使用了各种修饰符，`@PathVariable` 表示参数是从路径中得来的，而 `@RequestBody` 表示参数应该从 Http Request 的`body` 中解析，类似的 `@RequestHeader` 表示参数是 Http Request 的 Header 中定义的。
 3. `login` 方法返回的直接是对象，而 `register` 方法返回的是 `ResponseEntity` 。那么这两种写法有哪些区别？首先 `login` 直接返回对象的话，其实也是应用了 `@ResponseBody` ，Spring 会自动帮我们把返回的对象构建成 json 放到 Response 中。而 `ResponseEntity` 就可以更全面的定制自己想要返回的内容和状态，比如我们在某种情况下希望自己控制返回的状态码和返回的内容。这种时候使用 `ResponseEntity` 就非常方便。也就是说如果你感觉 Spring 默认的处理就很好，那么使用直接的对象返回没有任何问题。但如果你想自己有对状态和内容的控制，那么就可以使用 `ResponseEntity` 。

上面的代码中，我们看到有几个方法（ `findOneByLogin` 、 `findOneByMobile` 以及 `findOneByEmail` ）在 `UserRepo` 中是没有的。下面我们会来添加这几个方法，也顺便介绍 Spring Data MongoDB 中的查询方式。
