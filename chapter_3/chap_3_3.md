# MongoDB 支撑的 API

一般来说，做后端的话，数据库是必不可少的，我们提供的 API 很多也是需要查询或修改数据库中的某些数据的。数据库这些年的发展也是日新月异，但近 10 年来最显著的变化之一应该是 NoSQL 数据库的兴起了。

## 什么是 NoSQL？

NoSQL 是 **Not Only SQL** 的缩写，而不是许多人印象中的 No SQL，其实更正式的名字应该称作非关系型数据库。关系型数据库大家都很熟知了，比如 MySQL, Oracle, SQL Server 等，而 NoSQL 数据库泛指不能划归到关系型数据库中的其他数据库。这个类别可是很广泛的，也就是说即使都是 NoSQL 数据库，差异还是很大的。下表列出了一些典型的 NoSQL 数据库和它们的分类，当然类型的划分只是一个大概的参考，它们之间没有绝对的分界，也有交差的情况。

类型 | 典型数据库 | 特点
---------|----------|---------
 列存储 | Hbase、Cassandra | 按列存储数据，特点是方便存储结构化和半结构化数据，方便做数据压缩，对针对某一列或者某几列的查询有非常大的 I/O 优势。
 文档存储 | MongoDB、CouchDB | 用类似 `json` 的格式存储，存储的内容是文档型的。这样也就有有机会对某些字段建立索引，实现关系数据库的某些功能。
 键值存储 | MemcacheDB、Redis | 键值对形式，通过`key` 快速查询到其 `value`。
 图存储 | Neo4J | 图形关系的最佳存储。

### 为什么要使用 NoSQL？

要牢记的一点是，NoSQL 数据库并没有带来新的功能，传统关系型数据库其实从功能角度完全可以支持任何业务场景。NoSQL 兴起的原因是在某些领域，关系型数据库遇到了性能瓶颈。传统关系型一般在遇到性能问题时都会采用分表分库、主从复制、异构复制等等。但花费的精力和投入的资源越来越大，这正是 NoSQL 要解决的问题。

#### NoSQL 的优势

* 大数据量，高性能 -- NoSQL 数据库都具有非常高的读写性能，尤其在大数据量下，同样表现优秀。这得益于它的无关系性，数据库的结构简单。
* 灵活的数据模型 -- NoSQL 无需事先为要存储的数据建立字段，随时可以存储自定义的数据格式。而在关系数据库里，增删字段是一件非常麻烦的事情。如果是非常大数据量的表，增加字段简直就是一个噩梦。
* 高可用 - NoSQL 在不太影响性能的情况，就可以方便的实现高可用的架构。

### SQL or NoSQL？

需要注意的一点是，数据库在 SQL 和 NoSQL 之间非此即彼的取舍，相反，NoSQL 数据库与 SQL 数据库往往并存于企业的数据架构中。所以从使用角度来说，要分析业务场景，哪些适合关系型数据库，哪些适合 NoSQL。

NoSQL 和关系型数据库结合使用 -- 举例来说，业务系统如果可以使用关系型数据库，而且没有性能问题就不必使用 NoSQL，但现在客户需要一些用户行为分析的特性，而这个部分我们可以确定以后需要统计的维度越来越多，而且需要进行海量数据的查询，那么采用 NoSQL 就可以很好的进行这一块业务的设计，而关系型数据库也节省了大量 I/O。

NoSQL 替代关系型数据库 -- 假设说我们现在要构建一个社交网络，用户可以发布状态信息（信息内容包括文本、视频、音频和图片等）。那么我们可以画出一个下图的表关系结构。

![一个简单的社交网络的关系图](/assets/2018-04-02-15-32-50.png)

这种情况下我们想一下这样一个状态的结构怎么在页面中显示，如果我们希望显示状态的文字，以及关联的图片、音频、视频、用户评论、赞和用户的信息的话，我们需要关联八个表取得自己想要的数据。如果我们有这样的状态列表，而且是随着用户的滚动动态加载，同时需要监听是否有新内容的产生。这样一个任务我们需要太多这种复杂的查询了。

NoSQL 中的文档型数据库解决这类问题的思路是，干脆抛弃传统的表结构，你不是一个结构关系吗，那我就直接存储和传输一个这样的数据给你，像下面那样。

```json
{
    "id":"5894a12f-dae1-5ab0-5761-1371ba4f703e",
    "title":"2017年的Spring发展方向",
    "date":"2017-01-21",
    "body":"这篇文章主要探讨如何利用Spring Boot集成NoSQL",
    "createdBy":User,
    "images":["http://dev.local/myfirstimage.png","http://dev.local/mysecondimage.png"],
    "videos":[
        {"url":"http://dev.local/myfirstvideo.mp4", "title":"The first video"},
        {"url":"http://dev.local/mysecondvideo.mp4", "title":"The second video"}
    ],
    "audios":[
        {"url":"http://dev.local/myfirstaudio.mp3", "title":"The first audio"},
        {"url":"http://dev.local/mysecondaudio.mp3", "title":"The second audio"}
    ]
}

```

NoSQL 一般情况下是没有严格的 Schema 约束的，这也给开发带来较大的自由度。因为在关系型数据库中，一旦 Schema 确定，以后更改 Schema ，维护 Schema 是很麻烦的一件事。但反过来说 Schema 对于维护数据的完整性是非常必要的。

一般来说，如果你在做一个Web、物联网等类型的项目，你应该考虑使用NoSQL。如果你要面对的是一个对数据的完整性、事务处理等有严格要求的环境（比如财务系统），你应该考虑关系型数据库。

### 如何选择 NoSQL 数据库？

这么多类型的 NoSQL 数据库，到底怎么选择呢？影响选择的因素有很多，而选择也可能包含多种数据库，一般要分析具体的业务场景和需求：

1. 数据结构特点 -- 包括结构化、半结构化、字段是否可能变更、是否有大文本字段、数据字段是否可能变化等等。
2. 数据写入的特点 -- 包括插入（insert）和更新（update）比例、是否经常更新数据的某一部分，是否有原子更新要求等。
3. 数据查询特点 -- 包括查询的条件、查询的范围。

## MongoDB 的集成

MongoDB 是一个分布式的文档型的数据库，由 `C++` 语言编写，旨在提供可扩展的高性能数据存储解决方案。是非关系数据库当中功能比较丰富，比较像关系数据库的，因此也是 NoSQL 中应用较广泛的一种。

### 安装 MongoDB

安装 MongoDB 的话，你当然可以去官网下载，手动安装。但本书中建议使用 docker 进行安装。这样做的优点是，你可以轻松管理多个版本的 MongoDB，比如这个项目你使用了 MongoDB 2.2，那个项目使用了 MongoDB 3.6，如果采用本地安装的话，你可能比较会比较头疼了，但使用 docker 的话就安全没有压力。

```bash
# 直接抓取最新的 mongodb 镜像
docker pull mongo
# 和上面的效果一样，显式声明了版本
docker pull mongo:latest
# 拉取 3.6.3 版本的 mongodb
docker pull mongo:3.6.3
```

通过 `docker images` 命令可以查看已经拉取的镜像列表，下面的输出就是笔者机器上执行该命令后的结果。

```bash
REPOSITORY                            TAG                     IMAGE ID            CREATED             SIZE
gtm_nginx                             latest                  522c49bf937c        42 hours ago        17.9MB
nginx                                 latest                  7f70b30f2cc6        11 days ago         109MB
elasticsearch                         2.4.6                   cb429ea8b6e8        2 weeks ago         574MB
memcached                             latest                  e4aa3d5e67a3        4 weeks ago         60.8MB
wordpress                             latest                  80a6fca6cc6a        6 weeks ago         407MB
rabbitmq                              3.7-management-alpine   21d4ff1bda19        2 months ago        85.5MB
openjdk                               8-jre-alpine            b1bd879ca9b3        2 months ago        82MB
nginx                                 1.13.8-alpine           bb00c21b4edf        2 months ago        16.8MB
node                                  8-alpine                7c2983dfbf98        2 months ago        68.1MB
mongo                                 3.4.10                  e905a87e116d        3 months ago        360MB
openjdk                               8u121-jdk-alpine        630b87931295        10 months ago       101MB
```

拉取镜像之后，你就可以根据镜像建立并启动容器了，为了避免麻烦，我们还是映射 MongoDB 默认端口 27017 到本地的 27017。

```bash
# 根据 mongodb 3.6.3 镜像版本建立一个叫做 mymongo 的容器
docker run -p 27017:27017 --name mymongo -d mongo:3.6.3
```

如果要测试一下的话，可以通过以下命令登录到容器

```bash
docker exec -it mymongo bash
```

此时你应该可以看到终端的提示符改变了，应该变成了类似 `root@a0b609105000:` 这种，这说明你已经登录了容器中。在容器中执行 `mongo` ，可以看到如下输出

```bash
MongoDB shell version v3.4.10
connecting to: mongodb://127.0.0.1:27017
MongoDB server version: 3.4.10
Welcome to the MongoDB shell.
For interactive help, type "help".
For more comprehensive documentation, see
	http://docs.mongodb.org/
Questions? Try the support group
	http://groups.google.com/group/mongodb-user
Server has startup warnings:
2018-04-02T10:09:55.572+0000 I STORAGE  [initandlisten]
2018-04-02T10:09:55.572+0000 I STORAGE  [initandlisten] ** WARNING: Using the XFS filesystem is strongly recommended with the WiredTiger storage engine
2018-04-02T10:09:55.572+0000 I STORAGE  [initandlisten] **          See http://dochub.mongodb.org/core/prodnotes-filesystem
2018-04-02T10:09:56.017+0000 I CONTROL  [initandlisten]
2018-04-02T10:09:56.017+0000 I CONTROL  [initandlisten] ** WARNING: Access control is not enabled for the database.
2018-04-02T10:09:56.018+0000 I CONTROL  [initandlisten] **          Read and write access to data and configuration is unrestricted.
2018-04-02T10:09:56.018+0000 I CONTROL  [initandlisten]
```

接下来我们看看有哪些数据库，输入 `show databases`，不出意外的话，应该可以看到 2 个默认内置的数据库。

```log
admin  0.000GB
local  0.000GB
```

至此，我们的 MongoDB 测试成功，敲 `exit` 退出容器。

### 配置 Spring Boot 使用 MongoDB

Spring 的应用一般通过外部配置文件，比如 `*.properties` 或者 `*.yml` 文件来对应用进行配置。 Spring 默认情况下寻找配置文件会依照以下顺序进行

* 当前目录下的 `config` 文件夹
* 当前目录
* 位于 `classpath` 中的命名为 `config` 的包
* `classpath` 的根路径。而一个典型的 web 应用的资源 classpath 是如下位置，所以一般我们习惯把 `*.properties` 或者 `*.yml` 放在 `src/main/resources` 目录下。
  * `/META-INF/resources/`
  * `/resources/`
  * `/static/`
  * `/public/`

下面我们就在 `src/main/resources` 中建立一个 `application.yml` ，并在其中指定要连接的 MongoDB 的数据库名称。题外话，其实不指定这个数据库名称，Spring Boot 也可以正常启动并连接 MongoDB ，只不过这个时候连接的数据库默认是 `test` ，而这种方式在正式开发中我们是要避免的。

`yml` 这种格式非常简单，是以缩进来表示层级关系，比如我们要给 `spring.data.mongodb.database` 这个属性设置为 `gtm-api` ，也就是设置数据库名称为 `gtm-api` ，就可以通过下面的 `yml` 完成。

```yml
# application.yml
spring:
  data:
    mongodb:
      database: gtm-api
```

这个数据库现在还是不存在的，但是当有第一次有写操作的时候，这个 `gtm-api` 会自动被建立。

### 改写 Task

在我们刚刚的项目中集成 MongoDB 简单到令人发指，只需更改一下子项目 `api` 的 `build.gradle`，增加 2 个依赖：

```groovy
implementation("org.springframework.boot:spring-boot-starter-data-rest")
implementation("org.springframework.boot:spring-boot-starter-data-mongodb")
```

是的，就这么简单，这就是 Spring Boot 给开发者提供的开箱即用的体验。当然这个只是集成，但要使用的话，我们还得做一点工作，当然也很简单。

首先需要 Task 这个领域对象需要做一点小更改，在原有的 `Id` 属性前加一个 `@Id` 的注解，以标识这个字段是数据库中的 Id 字段。那么更改后的 Task 如下：

```java
package dev.local.gtm.api.domain;

import org.springframework.data.annotation.Id;

public class Task {
    @Id
    private String id;
    private String desc;
    private boolean completed;

    public String getId() {
        return id;
    }

    public void setId(String id) {
        this.id = id;
    }

    public String getDesc() {
        return desc;
    }

    public void setDesc(String desc) {
        this.desc = desc;
    }

    public boolean isCompleted() {
        return completed;
    }

    public void setCompleted(boolean completed) {
        this.completed = completed;
    }
}
```

领域对象改完，我们得有数据访问啊，所以建立一个和 `domain` 平行的 `package`，叫 `repository`。然后在这个下面建一个接口叫 `TaskRepo`

```java
package dev.local.gtm.api.repository;

import dev.local.gtm.api.domain.Task;
import org.springframework.data.mongodb.repository.MongoRepository;
import org.springframework.data.rest.core.annotation.RepositoryRestResource;

@RepositoryRestResource(collectionResourceRel = "tasks", path = "tasks")
public interface TaskRepo extends MongoRepository<Task, String> {
}
```

这个接口非常简单，首先继承了 `MongoRepository` ,这样一个简单的继承的能量大到让你吃惊，常见的增、删、改、查包括分页、排序等等，完全不用写一行代码。

而 `@RepositoryRestResource` 这个注解注意一下， `collectionResourceRel` 指定了集合资源的名称，怎么理解呢？好比说，我们现在要得到 `tasks` 的列表，访问 API 后返回的 `json` 中会有 `tasks: [...]` ，这个 `tasks` 就是我们在 `collectionResourceRel` 的值。如果我们改成 `collectionResourceRel = "todos"` ，那么返回的 json 中就变成了 `todos: [...]` 。而 `path` 指的是这个资源的相对路径， `path = "tasks"` 就会为这个 `respository` 生成一个 API 路径 `http://localhost:8080/tasks` 。

看到这里你应该也知道了，`TaskController` 可以下岗了，因为有能力更强的人顶替了它的作用，那么我们就删掉 `TaskController.java`。然后先终止之前启动的应用，如果你在 `terminal` 中启动的就按 `CTRL-C`，如果是 IDE 那么就按停止按钮，然后重新启动应用。

此时我们测试一下 API，还是在 Postman 中输入 `http://localhost:8080/tasks`，可以看到如下图类似的输出

![Tasks 的 GET 请求](/assets/2018-04-02-19-31-57.png)

API 是可以访问的，但返回的 `json` 的样子看起来有点奇怪，这个其实是一个符合 `HyperMedia` 的 Rest API 的返回结果，是一个 Rest 之上的进一步的封装标准，又叫 HATEOAS，当然又是一个缩写词： Hypermedia As The Engine Of Application State。那么究竟什么叫 `HATEOAS` 呢？我们一起往下看。

### HATEOAS

简单来说它是可以让客户端清晰的知道自己可以做什么，而无需依赖服务器端指示你做什么。原理呢，也很简单，通过返回的结果中包括不仅是数据本身，也包括指向相关资源的链接。拿上面的例子来说（虽然这种默认状态生成的东西不是很有代表性）：links 中有一个 profiles ，我们看看这个 profile 的链接 `http://localhost:8080/profile/tasks` 执行的结果是什么：

```json
{
    "alps": {
        "version": "1.0",
        "descriptors": [
            {
                "id": "task-representation",
                "href": "http://localhost:8080/profile/tasks",
                "descriptors": [
                    {
                        "name": "desc",
                        "type": "SEMANTIC"
                    },
                    {
                        "name": "completed",
                        "type": "SEMANTIC"
                    }
                ]
            },
            {
                "id": "create-tasks",
                "name": "tasks",
                "type": "UNSAFE",
                "rt": "#task-representation"
            },
            {
                "id": "get-tasks",
                "name": "tasks",
                "type": "SAFE",
                "rt": "#task-representation",
                "descriptors": [
                    {
                        "name": "page",
                        "doc": {
                            "value": "The page to return.",
                            "format": "TEXT"
                        },
                        "type": "SEMANTIC"
                    },
                    {
                        "name": "size",
                        "doc": {
                            "value": "The size of the page to return.",
                            "format": "TEXT"
                        },
                        "type": "SEMANTIC"
                    },
                    {
                        "name": "sort",
                        "doc": {
                            "value": "The sorting criteria to use to calculate the content of the page.",
                            "format": "TEXT"
                        },
                        "type": "SEMANTIC"
                    }
                ]
            },
            {
                "id": "update-task",
                "name": "task",
                "type": "IDEMPOTENT",
                "rt": "#task-representation"
            },
            {
                "id": "patch-task",
                "name": "task",
                "type": "UNSAFE",
                "rt": "#task-representation"
            },
            {
                "id": "delete-task",
                "name": "task",
                "type": "IDEMPOTENT",
                "rt": "#task-representation"
            },
            {
                "id": "get-task",
                "name": "task",
                "type": "SAFE",
                "rt": "#task-representation"
            }
        ]
    }
}
```

这个对象虽然我们暂时不是完全的理解，但大致可以猜出来，这个是 Task API 的元数据描述，告诉我们这个 API 中定义了哪些操作和接受哪些参数等等。

其实呢，Spring 是使用了一个叫 `ALPS` <http://alps.io/spec/index.html> 的专门描述应用语义的数据格式。摘出下面这一小段来分析一下，这个描述了一个 get 方法，类型是 `SAFE` 表明这个操作不会对系统状态产生影响（因为只是查询），而且这个操作返回的结果格式定义在 `task-representation` 中了。

```json
{
  "id": "get-task",
  "name": "task",
  "type": "SAFE",
  "rt": "#task-representation"
}
```

还是不太理解？没关系，我们再来做一个实验，我们用 Postman 构建一个 POST 请求，首先如下图所示，添加一个键值对到 Headers 中，当然别忘了选择左侧的下拉框中的 POST 方法

![为 POST 请求添加 Header](/assets/2018-04-02-19-30-12.png)

然后点击 `Body` 选择 `raw`，写如下的 `json`，当作请求体

![写一个 json 数据作为请求体](/assets/2018-04-02-19-46-47.png)

点击 Send 按钮，然后你应该可以看到如下输出

![Post 之后的返回](/assets/2018-04-02-19-51-42.png)

我们可以看到返回的 `links` 中包括了刚刚新增的 Task 的 API 链接 `http://localhost:8080/tasks/5ac21918446d71e6f417f225` （ `5ac21918446d71e6f417f225` 就是数据库自动为这个 Task 自动生成的 Id ），这样客户端可以方便的知道指向刚刚生成的 Task 的 API 链接。

我们可以试一下，在 Postman 中点击这个返回结果中的链接，Postman 会自动为这个链接新建一个 tab，点击 Send，就可以得到刚刚创建的 Task 了

![得到刚才生成的 Task](/assets/2018-04-02-19-58-47.png)

再举一个现实一些的例子，我们在开发一个“我的”页面时，一般情况下除了取得我的某些信息之外，因为在这个页面还会有一些可以链接到更具体信息的页面链接。如果客户端在取得比较概要信息的同时就得到这些详情的链接，那么客户端的开发就比较简单了，而且也更灵活了。

其实这个描述中还告诉我们一些分页的信息，比如每页 20 条记录( `size: 20` )、总共几页（ `totalPages：1` ）、总共多少个元素（ `totalElements: 1` ）、当前第几页（ `number: 0` ）。当然你也可以在发送 API 请求时，指定 `page` 、`size` 或 `sort` 参数。比如 `http://localhost:8080/tasks?page=0&size=10` 就是指定每页 10 条，当前页是第一页（从 0 开始）。

#### HAL 浏览器

使用 Postman 查看这种 API 的返回还是有点不直观，那么我们可以在 `build.gradle` 中添加一个依赖

```groovy
implementation("org.springframework.data:spring-data-rest-hal-browser")
```

然后重新启动应用，访问 <http://localhost:8080/browser/index.html#/>，然后就可以使用这个 HAL 浏览器去实验和观察 API 的请求和返回。


![HAL 浏览器](/assets/2018-04-02-23-12-46.png)

## 魔法的背后

这么简单就生成一个有数据库支持的 REST API，这件事看起来比较魔幻，但一般这么魔幻的事情总感觉不太托底，除非我们知道背后的原理是什么。首先再来回顾一下 `TaskRepo` 的代码：

```java
@RepositoryRestResource(collectionResourceRel = "tasks", path = "tasks")
public interface TaskRepo extends MongoRepository<Task, String> {
}
```

Spring 是最早的几个 IoC（控制反转或者叫 DI ）框架之一，所以最擅长的就是依赖的注入了。这里我们写了一个 Interface ，可以猜到 Spring 一定是有一个这个接口的实现在运行时注入了进去。如果我们去 `spring-data-mongodb`  的源码中看一下就知道是怎么回事了，这里只举一个小例子，大家可以去看一下 `SimpleMongoRepository.java` <https://github.com/spring-projects/spring-data-mongodb/blob/master/spring-data-mongodb/src/main/java/org/springframework/data/mongodb/repository/support/SimpleMongoRepository.java>，由于源码太长，我只截取一部分：

```java
public class SimpleMongoRepository<T, ID extends Serializable> implements MongoRepository<T, ID> {

    private final MongoOperations mongoOperations;
    private final MongoEntityInformation<T, ID> entityInformation;

    /**
        * Creates a new {@link SimpleMongoRepository} for the given {@link MongoEntityInformation} and {@link MongoTemplate}.
        *
        * @param metadata must not be {@literal null}.
        * @param mongoOperations must not be {@literal null}.
        */
    public SimpleMongoRepository(MongoEntityInformation<T, ID> metadata, MongoOperations mongoOperations) {

        Assert.notNull(mongoOperations);
        Assert.notNull(metadata);

        this.entityInformation = metadata;
        this.mongoOperations = mongoOperations;
    }

    /*
        * (non-Javadoc)
        * @see org.springframework.data.repository.CrudRepository#save(java.lang.Object)
        */
    public <S extends T> S save(S entity) {

        Assert.notNull(entity, "Entity must not be null!");

        if (entityInformation.isNew(entity)) {
            mongoOperations.insert(entity, entityInformation.getCollectionName());
        } else {
            mongoOperations.save(entity, entityInformation.getCollectionName());
        }

        return entity;
    }
    ...
    public T findOne(ID id) {
        Assert.notNull(id, "The given id must not be null!");
        return mongoOperations.findById(id, entityInformation.getJavaType(), entityInformation.getCollectionName());
    }

    private Query getIdQuery(Object id) {
        return new Query(getIdCriteria(id));
    }

    private Criteria getIdCriteria(Object id) {
        return where(entityInformation.getIdAttribute()).is(id);
    }
    ...
}
```

也就是说其实在运行时 Spring 将这个类或者其他具体接口的实现类注入了应用。这个类中有支持各种数据库的操作。一般的应用了解到这步就可以了，有兴趣的同学可以继续深入研究。

虽然不想在具体类上继续研究，但我们还是应该多了解一些关于 `MongoRepository` 的东西。这个接口继承了 `PagingAndSortingRepository` （定义了排序和分页） 和 `QueryByExampleExecutor`。而 `PagingAndSortingRepository` 又继承了 `CrudRepository` （定义了增删改查等）。

第二个魔法就是它直接对 MongoDB 中的集合（本例中的 tasks ）映射到了一个REST URI（tasks）。因此我们连 Controller 都没写就把 API 搞出来了，而且还是个 Hypermedia REST。

其实呢，这个注解 `@RepositoryRestResource(collectionResourceRel = "tasks", path = "tasks")` 只在你有特殊需求的时候才需要，比如是否需要暴露这个 `repository` 到 REST 资源？是否需要重新定义这个资源的集合名称和单个名称（比如有时候某些单词的复数形式是特殊的，就需要指定一下 `collectionResourceRel` 和 `path` ）。本例中如果我们不加 `@RepositoryRestResource` 这个注解的话，同样也可以生成 API，只不过其路径按照默认英语复数定义路径，也就是说如果我们不想按默认路径的话，指定这个注解可以自定义 API 路径，大家可以试试把这个注解去掉，然后重启服务，访问 `http://localhost:8080/tasks` 看看。

### REST 的简单介绍

REST 简单来说可以理解成几个基本规则

1. 以资源名称构成 API 路径，也就是定义资源尽量使用名词
2. 以动词定义要对资源执行的操作，也就是使用不同的请求方法（GET/POST/PUT/DELETE 等)访问资源路径就代表着要对资源采用什么操作。一般 GET 表示查询， POST 代表新增，PUT 表示更改，而 DELETE 表示删除。
3. 资源的复数形式表示列表，而列表后跟资源唯一标识表示取得具体列表中的某一资源。

#### 几个例子

比如 `/api/users` 代表了 `users` 这个资源，

* 那么一般来说如果以 `GET` 方法访问 `/api/users` 表示要得到用户的列表。此处需要注意，列表路径是复数形式，如果没做特殊处理， Spring 会按照英语的语法给出复数形式。
* 以 `POST` 方法访问 `/api/users` 表示要新增一个 `user` ，这个要新增的用户信息一般以 `json` 形式作为 `Request Body` 上传。
* 那么如何得到某一个用户呢？使用 `GET` 方法访问 `/api/users/:id` ，比如如果用户 `ID` 是 `1` ，那么得到这个用户的 `URL` 是 `/api/users/1` 。
* 对该用户的更新是通过 `PUT` 或 `PATCH` 方法访问 `/api/users/:id` 。同样的修改的信息以 `json` 形式作为 `Request Body` 上传。 `PUT` 和 `PATCH` 的区别在于 `PUT` 要求上传该对象的完整形式，而且更新也是全部进行更新。而 `PATCH` 是可以上传部分改变的信息，比 `PUT` 要节省传输带宽。
* 删除用户当然也就是 `DELETE` 方法 `/api/users/:id` 。

#### REST 的成熟度

REST API 有一个 `Richardson Maturity Model` 的评价标准

* Level 0 （Swamp of POX）：这一级其实一点都不符合 REST，所以是 `Level 0` ，也就是你使用了 HTTP 或者类似的协议，使用某一种 HTTP 方法，多数情况下是 `POST` 。
* Level 1 （Resources）：这一级你开始定义资源，也就是说对于不同的资源，你有不同的 URL 标识，但仍是使用 HTTP 的某一种方法，多数情况是 POST
* Level 2 （HTTP verbs）: 这一级除了资源的划分，你也要是用不同的方法实现不同的对资源的操作，比如 POST 代表新增，PUT 代表修改，DELETE 代表删除而 GET 表示查询。此外需要使用协议规定的响应码标识不同的状态，比如在出现错误时返回 `200` 就是一个错误用法。
* Level 3 （Hypermedia controls）：这一级就是客户端可用自己发现我们的 API，我们上面做出的 `HATEOAS` 类型的 API 就是符合 Level 3 的。但很可惜的是在现实开发中，目前达到这一级别的团队很少。

## 让后端也能热更新

热更新顾名思义就是在不停止服务的情况下，将代码的改变立即部署应用到程序中。前端工程领域大部分框架都标配了热更新，这样使得开发的体验非常好。其实后端领域 `node.js` 和一些 `python` 、`ruby` 框架也是支持热更新的 那么 JAVA 后端可不可以呢？Spring Boot 作为一个开箱即用的框架当然会考虑到这一点。

### Spring Boot DevTools

所以我们只需要在 `api` 子项目的 `build.gradle` 中加入一行 `implementation("org.springframework.boot:spring-boot-devtools")` 即可实现热更新了。当然由于 JAVA 不是一个动态语言，所以这个“热更新”其实是重启服务。但 Spring Boot Devtools 提供的“重启”不是冷启动，而是通过两个类加载器，文件没有改变的作为“基线类加载器”，你更新的文件作为“重启类加载器”。只有“重启类加载器”是一次性使用的，用后就丢弃掉，“基线类加载器”并不会每次重新创建，所以这比冷启动要快很多。

```groovy
apply plugin: 'org.springframework.boot'

dependencies {
    implementation("org.springframework.boot:spring-boot-devtools")
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-data-rest")
    implementation("org.springframework.boot:spring-boot-starter-data-mongodb")
    implementation("org.springframework.data:spring-data-rest-hal-browser")
}
```

在 IDEA 中，右键选择 Application，然后选择 `Run Application` 就可以了，现在你尝试更新一些文件，在 console 中就可以看到服务自动重启了。

![在 IDEA 中启动 Spring Boot 应用](/assets/2018-04-04-13-13-58.png)

Spring Boot 默认对 `classpaht` 下的文件更新会触发重启，但对于静态资源，一般情况下不会引起热更新，Spring Boot 默认对以下文件夹的变化不会触发更新 `/META-INF/maven`, `/META-INF/resources`, `/resources`, `/static`, `/public`, 和 `/templates` 。你可以通过调整 `spring.devtools.restart.exclude` 属性来决定哪些文件被排除在更新触发器之外。你同样也可以指定哪些不再 `classpath` 下的文件会触发更新，只需设置属性 `spring.devtools.restart.additional-paths` 即可。

对于静态资源，Spring Boot 提供了 `LiveReload` 的方式进行热更新。但这个在我们的前后端分离项目中没有什么意义，所以就不在这里阐述了，有兴趣的同学可以去 <https://docs.spring.io/spring-boot/docs/current/reference/html/using-boot-devtools.html> 看官方文档。
