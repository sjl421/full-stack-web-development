# Spring Data 中的查询

Spring Data 是 Spring 的一个子项目 <https://projects.spring.io/spring-data/> ，这个项目的目标是要简化对于不同数据库（或者更广泛的叫做不同的数据持久化存储方式）的操作以及形成一致的数据访问层的抽象。

## 基础概念 -- Repository

Spring Data 的核心是 `Repository` ，这个单词译成中文是 `存储库` ，感觉还是不太达意，所以后面提到时还是使用英文原文。 `Repository` 作为 Spring Data 中最基础的抽象。

```java
@Indexed
public interface Repository<T, ID> {
}
```

这个抽象简单到爆，其中 `T` 是领域对象的类型， `ID` 是领域对象存储在数据库中的 ID。其他的 Repository 都是继承了这个基础接口。当然从这个接口的定义我们也会知道，不管使用的是哪种数据库，如果要定义一个 repository 就肯定要提供领域对象和 ID 的类型。

### 几个内建的 Repository

在 Spring Data 中，我们经常会遇到以下几位

* `CrudRepository` -- `Crud` 就是增删改查的缩写，即 Create, Read, Update, Delete 的首字母连在一起。顾名思义就是定义了基础的增删改查操作。
* `PagingAndSortingRepository` -- 也是命名上就可以看出是定义了分页和排序的操作。它是继承了 `CrudRepository` 之后又提供了分页和排序的操作。
* `MongoRepository` -- 和上面两个不同，MongoRepository 是一个和数据库相关的接口，它继承了 `PagingAndSortingRepository` 同时封装了一些 MongoDB 特有的方法。


需要指出的是 `CrudRepository` 和 `PagingAndSortingRepository` 是通用的，也就是说在 `Spring Data Jpa` 和 `Spring Data MongoDB` 中都是一致的，不是受限于某一个数据库。

从 `CrudRepository` 的源码中可以看到基础的增删改查方法都已经在这里定义好了。

```java
public interface CrudRepository<T, ID extends Serializable> extends Repository<T, ID> {

  // 保存
  <S extends T> S save(S entity);
  // 返回指定 ID 对应的对象
  Optional<T> findById(ID primaryKey);
  // 查找所有对象
  Iterable<T> findAll();
  // 返回数量
  long count();
  // 删除实体
  void delete(T entity);
  // 判断指定的 ID 是否存在
  boolean existsById(ID primaryKey);

  // 省略其他部分
}
```

而 `PagingAndSortingRepository` 在此之上提供了排序和分页的方法。

```java
public interface PagingAndSortingRepository<T, ID> extends CrudRepository<T, ID> {

  Iterable<T> findAll(Sort sort);

  Page<T> findAll(Pageable pageable);
}
```

我们可以从 `MongoRepository` 的源码中看到它增加了 `insert` 方法，这个方法就是不在通用的接口中，而是在 MongoDB 的专属接口中定义的。

```java
public interface MongoRepository<T, ID> extends PagingAndSortingRepository<T, ID>, QueryByExampleExecutor<T> {

  @Override
  <S extends T> List<S> saveAll(Iterable<S> entites);

  @Override
  List<T> findAll();

  @Override
  List<T> findAll(Sort sort);

  <S extends T> S insert(S entity); // <--- 数据库相关的方法

  <S extends T> List<S> insert(Iterable<S> entities); // <--- 数据库相关的方法

  @Override
  <S extends T> List<S> findAll(Example<S> example);

  @Override
  <S extends T> List<S> findAll(Example<S> example, Sort sort);
}
```

## 查询方式

Spring Data 内建提供了动态依赖方法命名构建查询的方法。这个方式粗听上去感觉可能是个玩具，复杂查询应付不了吧？但恰恰相反，这个方式的灵活性非常好，基本可以应对较复杂的查询。

那么我们就来增加一个需求来试试，我们可以通过查询 Task 的描述中的关键字来搜索符合的项目。

显然这个查询不是默认的操作，那么这个需求在 Spring Boot 中怎么实现呢？非常简单，只需在 `TaskRepo` 中添加一个方法：

```java
...
@Repository
public interface TaskRepo extends MongoRepository<Task, String> {
    Page<Task> findByDescLike(Pageable pageable, @Param("desc") String desc);
}
```

然后新建 `/api/web/rest/TaskResource.java` ，我们创建一个 GET 方法 `getAllTasks` ，这个对应的 URL 是 `/api/tasks` ，现在我们让它可以携带参数，也就是 `/api/tasks?desc=xxx` 这种形式。在方法中我们声明这个 `@RequestParam` 的参数 `desc` 。如果这个参数不存在，那么就返回全部列表，否则就执行 `findByDescLike` 的查询。

```java
@Log4j2
@RestController
@RequestMapping("/api")
@RequiredArgsConstructor
public class TaskResource {
    private final TaskRepo taskRepo;

    @GetMapping("/tasks")
    public List<Task> getAllTasks(Pageable pageable, @RequestParam(required = false) String desc) {
        log.debug("REST 请求 -- 查询所有 Task");
        return desc == null ?
                taskRepo.findAll(pageable).getContent() :
                taskRepo.findByDescLike(pageable, desc).getContent();
    }
}
```

让我们到 Postman 中试一下，看看结果，先是看看不带参数，返回了全部列表。

![不带 desc 参数的请求](/assets/2018-04-17-16-26-14.png)

再试试带上参数 `?desc=you` ，这个关键字是存在的，返回的是符合条件的列表

![存在匹配关键字的请求](/assets/2018-04-17-16-28-04.png)

如果参数改为 `?desc=me` ，这个关键字就不存在了，所以返回空列表。

![不存在匹配关键字的请求](/assets/2018-04-17-16-30-37.png)

你说这里肯定有鬼，我同意。那么我们试试把这个方法改个名字 `findDescLike` ，果然不好用了。为什么呢？这套神奇的疗法的背后还是那个我们提到的 `Convention over configuration` ，要神奇的疗效就得遵循 Spring 的配方。这个配方就是方法的命名是有讲究的：Spring 提供了一套可以通过命名规则进行查询构建的机制。这套机制会把方法名首先过滤一些关键字，比如 `find…By` 、 `read…By` 、 `query…By` 、 `count…By` 和 `get…By` 等。系统会根据关键字将命名解析成 2 个子语句，第一个 `By` 是区分这两个子语句的关键词。这个 `By` 之前的子语句是查询子语句（指明返回要查询的对象），后面的部分是条件子语句。如果直接就是 `findBy…` 返回的就是定义 Respository 时指定的领域对象集合（本例中的 Task 组成的集合）。

一般到这里，有的同学可能会问 `find…By`, `read…By`, `query…By`,  `get…By` 到底有什么区别啊？答案是。。。木有区别，就是别名，从下面的源文件 <https://github.com/spring-projects/spring-data-commons/blob/master/src/main/java/org/springframework/data/repository/query/parser/PartTree.java> 定义中可以看到这几个东东其实生成的查询是一样的，这种让你不用查文档都可以写对的方式也比较贴近目前流行的自然语言描述风格（类似各种 DSL）。

```java
private static final String QUERY_PATTERN = "find|read|get|query|stream";
```

刚刚我们实验了模糊查询，那如果要是精确查找怎么做呢，比如我们要筛选出已完成或未完成的 Task ，也很简单：

```java
List<Task> findByCompleted(@Param("completed") boolean completed);
```

## 复杂类型查询

看到这里你会问，这都是简单类型，如果复杂类型怎么办？嗯，好的，我们还是增加一个需求看一下：现在需求是要这个 API 是多用户的，每个用户看到的 Task 都是他们自己创建的。所以我们改造一下 `Task` 让它有一个 `User` 类型的 `owner` 属性。

```java
@Data @Builder
public class Task {
    @Id
    private String id;
    private String desc;
    private boolean completed;
    private User owner;
}
```

然后给 `TaskRepo` 添加一个方法

```java
@Repository
public interface TaskRepo extends MongoRepository<Task, String> {
    Page<Task> findByDescLike(Pageable pageable, @Param("desc") String desc);
    Page<Task> findByOwnerMobile(Pageable pageable, @Param("mobile") String mobile);
}
```

然后可以通过 Postman 使用 `/api/tasks` 的 POST 方法添加几个数据，比如类似下面的

```json
{
  "desc": "11点30的午餐",
  "completed": false,
  "owner": {
    "id": "2",
    "login": "lisi",
    "mobile": "13012351235",
    "email": "lisi@local.dev",
    "name": "Li Si"
  }
}
```

现在我们如果执行一个 GET 请求到 <http://localhost:8080/api/tasks/search/findByUserMobile?mobile=13012351235> 就会得到下图的结果。

![手机号匹配返回的结果](/assets/2018-04-17-19-12-42.png)

我们来分析这个 `findByOwnerMobile` 是如何解析的：首先在 `By` 之后，解析器会按照 `camel` （每个单词首字母大写）的规则来分词。那么第一个词是 `Owner`，这个属性在 `Task` 中有没有呢？有的，但是这个属性是另一个对象类型 `User` ，所以紧跟着这个词的 `Mobile` 就要在 `User` 类中去查找是否有 `Mobile` 这个属性。聪明如你，肯定会想到，那如果在 `Task` 类中如果还有一个属性叫 `ownerMobile` 怎么办？是的，这种情况下 `ownerMobile` 会被优先匹配，此时请使用 `_` 来显性分词处理这种混淆。也就是说如果我们的 `Task` 类中同时有 `owner` 和 `ownerMobile` 两个属性的情况下，我们如果想要指定的是 `owner` 的 `mobile` ，那么需要写成 `findByOwner_Mobile`。

## 自定义查询

好吧，到现在我估计还有一大波攻城狮表示不服，实际开发中需要的查询比上面的要复杂的多，再复杂一些怎么办？还是用例子来说话吧，那么现在我们想要模糊搜索指定用户的 Task 中描述的关键字，返回匹配的集合。这个需求我们只需改动一行，这个以命名规则为基础的查询条件是可以加 `And` 、`Or` 这种关联多个条件的关键字的。

```java
List<Todo> findByOwnerMobileAndDescLike(@Param("mobile") String mobile, @Param("desc") String desc);
```

当然，还有其他操作符：`Between` (值在两者之间), `LessThan` (小于), `GreaterThan` （大于）, `Like` （包含）, `IgnoreCase` （忽略大小写）, `AllIgnoreCase` （对于多个参数全部忽略大小写）, `OrderBy` （引导排序子语句）, `Asc` （升序，仅在 `OrderBy` 后有效） 和 `Desc` （降序，仅在 `OrderBy` 后有效）。

刚刚我们谈到的都是对于查询条件子语句的构建，其实在 `By` 之前，对于要查询的对象也可以有限定的修饰词 `Distinct` （去重，如有重复取一个值）。比如有可能返回的结果有重复的记录，可以使用 `findDistinctTaskByOwnerMobileAndDescLike`。

简单是简单了，但遇到复杂查询的话，这个方法名不得上天了？我可以直接写查询语句吗？几乎所有码农都会问的问题。当然可以咯，也是同样简单，就是给方法加上一个元数据修饰符 `@Query`

```java
@Query("{ 'owner.mobile': ?0, 'desc': { '$regex': ?1} }")
    List<Task> searchTasks(@Param("mobile") String mobile, @Param("desc") String desc);
```

采用这种方式我们就不用按照命名规则起方法名了，可以直接使用 MongoDB 的查询进行。上面的例子中有几个地方需要说明一下

1. `?0` 和 `?1` 是参数的占位符，`?0` 表示第一个参数，也就是 `mobile`；而 `?1` 表示第二个参数也就是 `desc`。
2. MongoDB中没有关系型数据库的Like关键字，需要以正则表达式的方式达成类似的功能。
3. 如果要指定 Id 时，需要写成 `xxx._id` ，前面的下划线是要有的。

这样的查询已经相当于你在 MongoDB 中查询了，有兴趣的同学可以把下面的代码在 MongoDB 的控制台输入实验一下看是否好用。所以这种程度的支持基本可以让我们写出相对较复杂的查询了。

```js
db.task.find(
    {
        "owner.mobile": "13012351235",
        "desc": { "$regex": "11" }
    })
```

但肯定还是不够的，对于开发人员来讲，如果不给可以自定义的方式基本没人会用的，因为总有这样那样的原因会导致我们希望能完全掌控我们的查询或存储过程。


```java
package dev.local.gtm.api.repository;

import dev.local.gtm.api.domain.User;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.mongodb.repository.MongoRepository;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;

import java.util.Optional;

@Repository
public interface UserRepo extends MongoRepository<User, String> {
    String USERS_BY_LOGIN_CACHE = "usersByLogin";

    String USERS_BY_MOBILE_CACHE = "usersByMobile";

    String USERS_BY_EMAIL_CACHE = "usersByEmail";

    @Cacheable(cacheNames = USERS_BY_MOBILE_CACHE)
    Optional<User> findOneByMobile(@Param("mobile") String mobile);

    @Cacheable(cacheNames = USERS_BY_EMAIL_CACHE)
    Optional<User> findOneByEmail(@Param("email") String email);

    @Cacheable(cacheNames = USERS_BY_LOGIN_CACHE)
    Optional<User> findOneByLogin(@Param("login") String login);

    Page<User> findAllByLoginNot(Pageable pageable, @Param("login") String login);
}
```
