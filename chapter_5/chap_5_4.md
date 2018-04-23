# 构建安全的 API 接口

估计很多同学看了之前的登录和注册之后会吐槽，这个实现方式实在是不怎么样啊。是的，前面实现的鉴权方式的缺点有

1. 密码明文存储 -- 这个导致安全性问题就不用多说了。
2. 并未实现对 API 的保护 -- 换句话说登录与否和能否访问 API 没有关系，所以这是一个假登录。
3. 没有角色的划分 -- 所有的接口都是一视同仁的，但实际项目中肯定要有各种角色允许访问的接口是不一样的限制。

那么，接下来这节我们就一起学习如何使用 JWT 实现一个基于 `token` 的 API 鉴权方式以及使用 Spring Security 实现给予角色的权限控制。

## 为什么要保护 API？

通常情况下，把 API 直接暴露出去是风险很大的，不说别的，直接被机器攻击就喝一壶的。那么一般来说，对 API 要划分出一定的权限级别，然后做一个用户的鉴权，依据鉴权结果给予用户开放对应的 API 。目前，比较主流的方案有几种:

 1. 用户名和密码鉴权，使用 Session 保存用户鉴权结果。
 2. 自行采用 `token` 进行鉴权，自己设计的 `token` 往往由于设计时没有考虑周全，后期存在各种兼容性问题。
 3. 使用 `OAuth`/`OAuth2` 进行鉴权（其实 `OAuth` 也是一种基于 `token` 的鉴权，只是没有规定 `token` 的生成方式）
 4. 使用 `JWT` 作为 `token`

第一种就不介绍了，由于依赖 `Session` 来维护状态，也不太适合移动时代，新的项目就不要采用了。第二种的兼容性和可维护性较差而 `OAuth` 其实对于不做开放平台的公司有些过于复杂。我们主要介绍第四种：`JWT` 。

## 什么是JWT？

下面是一个 JWT 的工作流程图。模拟一下实际的流程是这样的（假设受保护的 API 在`/protected`中）

 1. 用户导航到登录页，输入用户名、密码，进行登录
 2. 服务器验证登录鉴权，如果改用户合法，根据用户的信息和服务器的规则生成 `JWT token`
 3. 服务器将该 `token` 以 json 形式返回（其实不一定要 json 形式，这里说的是一种常见的做法）
 4. 用户得到 token，存在 localStorage、cookie、IndexDB 或其它数据存储形式中。
 5. 以后用户请求`/protected`中的 API 时，在请求的 header 中加入 `Authorization: Bearer xxxx(token)` 。此处注意 token 之前有一个 7 字符长度的 `Bearer`，注意 `Bearer` 后应该有一个空格。
 6. 服务器端对此 token 进行检验，如果合法就解析其中内容，根据其拥有的权限和自己的业务逻辑给出对应的响应结果。
 7. 用户取得结果

![JWT工作流程图](/assets/2018-04-09-17-48-56.png)

为了更好的理解这个token是什么，我们先来看一个token生成后的样子，下面那坨乱糟糟的就是了。

```txt
eyJhbGciOiJIUzUxMiJ9.eyJzdWIiOiJ3YW5nIiwiY3JlYXRlZCI6MTQ4OTA3OTk4MTM5MywiZXhwIjoxNDg5Njg0NzgxfQ.RC-BYCe_UZ2URtWddUpWXIp4NMsoeq2O6UF-8tVplqXY1-CI9u1-a-9DAAJGfNWkHE81mpnR3gXzfrBAB3WUAg
```

但仔细看到的话还是可以看到这个 `token` 分成了三部分，每部分用 `.` 分隔，每段都是用 `Base64` 编码的。如果我们用一个Base64的解码器的话  <https://www.base64decode.org> ，可以看到第一部分 `eyJhbGciOiJIUzUxMiJ9` 被解析成了:

```json
{
    "alg":"HS512"
}
```

这是告诉我们HMAC采用HS512算法对JWT进行的签名。

第二部分 `eyJzdWIiOiJ3YW5nIiwiY3JlYXRlZCI6MTQ4OTA3OTk4MTM5MywiZXhwIjoxNDg5Njg0NzgxfQ` 被解码之后是

```json
{
    "sub":"wang",
    "created":1489079981393,
    "exp":1489684781
}

```

这段告诉我们这个 `token` 中含有的数据声明（ `Claim` ），这个例子里面有三个声明：`sub`, `created` 和 `exp`。在我们这个例子中，分别代表着用户名、创建时间和过期时间，当然你可以把任意数据声明在这里。

看到这里，你可能会想这是个什么鬼 `token` ，所有信息都透明啊，安全怎么保障？别急，我们看看 `token` 的第三段  `RC-BYCe_UZ2URtWddUpWXIp4NMsoeq2O6UF-8tVplqXY1-CI9u1-a-9DAAJGfNWkHE81mpnR3gXzfrBAB3WUAg` 。同样使用 Base64 解码之后，咦，这是什么东东

```txt
D X	DmYTeȧLUZcPZ0$gZAY_7wY@
```

最后一段其实是签名，这个签名必须知道秘钥才能计算。这个也是 `JWT` 的安全保障。这里提一点注意事项，由于数据声明（ `Claim` ）是公开的，千万不要把密码等敏感字段放进去，否则就等于是公开给别人了。

也就是说JWT是由三段组成的，按官方的叫法分别是 `header` （头）、 `payload` （负载）和 `signature` （签名）：

```txt
header.payload.signature
```

头中的数据通常包含两部分：一个是我们刚刚看到的 `alg`，这个词是 `algorithm` 的缩写，就是指明算法。另一个可以添加的字段是 `token` 的类型(按 `RFC 7519` 实现的 `token` 机制可不只 JWT 一种)，但如果我们采用的是 `JWT` 的话，指定这个就多余了。

```json
{
  "alg": "HS512",
  "typ": "JWT"
}
```

`payload` 中可以放置三类数据：系统保留的、公共的和私有的：

* 系统保留的声明（ `Reserved claims` ）：这类声明不是必须的，但是是建议使用的，包括： `iss` (签发者), `exp` (过期时间), `sub` (主题), `aud` (目标受众)等。这里我们发现都用的缩写的三个字符，这是由于 `JWT` 的目标就是尽可能小巧。
* 公共声明：这类声明需要在 IANA JSON Web Token Registry 中定义或者提供一个 URI ，因为要避免重名等冲突。
* 私有声明：这个就是你根据业务需要自己定义的数据了。

签名的过程是这样的：采用 `header` 中声明的算法，接受三个参数： `base64` 编码的 `header` 、 `base64` 编码的 `payload` 和秘钥（ `secret` ）进行运算。签名这一部分如果你愿意的话，可以采用 `RSASHA256` 的方式进行公钥、私钥对的方式进行，如果安全性要求的高的话。

```txt
HMACSHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  secret)
```

## JWT的生成和解析

为了简化我们的工作，这里引入一个比较成熟的 JWT 类库，叫 `jjwt` <https://github.com/jwtk/jjwt> 。这个类库可以用于 Java 和 Android 的 JWT token 的生成和验证。

首先需要在 `api/build.gradle` 中添加依赖

```groovy
implementation("io.jsonwebtoken:jjwt:0.9.0")
```

`JWT` 的生成可以使用下面这样的代码完成：

```java
String generateToken(Map<String, Object> claims) {
    return Jwts.builder()
            .setClaims(claims)
            .setExpiration(generateExpirationDate())
            .signWith(SignatureAlgorithm.HS512, secret) //采用什么算法是可以自己选择的，不一定非要采用HS512
            .compact();
}
```

数据声明（ `Claim` ）其实就是一个 `Map` ，比如我们想放入用户名，可以简单的创建一个 `Map` 然后 `put` 进去就可以了。

```java
Map<String, Object> claims = new HashMap<>();
claims.put(CLAIM_KEY_USERNAME, username());
```

解析也很简单，利用 `jjwt` 提供的 parser 传入秘钥，然后就可以解析 `token` 了。

```java
Claims getClaimsFromToken(String token) {
    Claims claims;
    try {
        claims = Jwts.parser()
                .setSigningKey(secret)
                .parseClaimsJws(token)
                .getBody();
    } catch (Exception e) {
        claims = null;
    }
    return claims;
}
```

`JWT` 本身没啥难度，但安全整体是一个比较复杂的事情， `JWT` 只不过提供了一种基于 `token` 的请求验证机制。但我们的用户权限，对于 API 的权限划分、资源的权限划分，用户的验证等等都不是 `JWT` 负责的。也就是说，请求验证后，你是否有权限看对应的内容是由你的用户角色决定的。所以我们这里要利用 `Spring` 的一个子项目 `Spring Security` 来简化我们的工作。在开始 Spring Security 的工作前，让我们对基于角色的权限的权限做一个简单的背景知识介绍。

## 权限的设计

### ACL 权限模型

`ACL` 是英文 `Access Control Lists` 的缩写， `ACL` 指的是对于某个数据对象的权限列表，一个 `ACL` 会指出哪些用户或系统进程被授予了对数据对象的访问权限，以及允许什么样的操作。比如文件的 `ACL` 通常是类似 `(ZhangSan: read, write; LiSi: read)` 。

### 基于 RBAC 的权限模型

`RBAC` 是 `Role-Based Access Control` 的英文缩写，翻译过来就是基于角色的访问控制。`RBAC` 认为权限授权实际上是 `Who` 、 `What` 、 `How` 决定的。在 `RBAC` 模型中，`Who` 、 `What` 、 `How` 构成了访问权限三元组，即 `Who` 对 `What` 进行 `How` 的操作。其中 `Who` 是权限的拥有者或主体（如：`User` 、 `Role` ）， `What` 是资源或对象（`Resource` 、 `Class`)

`RBAC` 主要分为四种变化形式：

* 核心模型 `RBAC-0`（ `Core RBAC` ）
* 角色分层模型 `RBAC-1`（ `Hierarchal RBAC` ）
* 角色限制模型 `RBAC-2`（ `Constraint RBAC` ）
* 统一模型 `RBAC-3`（ `Combines RBAC` ）

最重要也是最基本的是 `RBAC-0`，因为这个模型是最小化实现 `RBAC` 权限思想的方式，其他的都是在此基础上的补充和变化。

![RBAC 领域模型](/assets/2018-04-07-18-07-32.png)

`RBAC` 和 `ACL` 的区别在于 `RBAC` 将权限分配到对组织有意义的特定操作上而不是分配到底层的数据对象上。举个小例子，ACL 可以用来授予或拒绝某个系统文件的写访问请求，但它不能判断这个文件是怎样被更改的。

在 `RBAC` 系统中，一个操作可以是在一个财务系统中“创建一个账簿”或者在一个医疗系统中“执行一个血糖测试”。这些操作的权限分配对组织来讲是有意义的，因为组织内的这些操作是一个基本单位的流程。

`RBAC` 非常适合职责分离的需求，这种需求下经常会要求确保至少 2 个或 2 个以上的人员参与到授权的操作中去。其实一个最小化的 RBAC 模型和 `ACLg` （带分组的 `ACL` ）是等效的。

### 前后端的权限划分

一个系统前后端都会涉及到权限的处理，但后端的权限是更为重要一些，因为这涉及到系统的数据，也就是说后端如果不设防，那么前端的安全做的再好也没有用。但相反如果后端的安全性设计的较好，即使前端有些漏洞，也不会影响后端的数据。

通常一个前后端分离的系统，我们需要对后端的 API 进行保护，这种保护有多个层次

* token -- 这个是最基础的，就是验证你的请求是一个授权请求
* 角色 -- 即使有了合法的 token ，还要看你的角色是否可以访问某些 API。打个比方， token 类似于小区的大门钥匙，但进入你的家，你还得有家门钥匙。这个角色相当于是你是否有权进入这个房间。
* 访问频次 (rate limit) -- API 既然开放出来就是让用户使用的，但是我们不欢迎恶意的使用，比如有人写程序对你的鉴权接口进行字典攻击去得到用户的密码，如果没有限制，他们这么试下去，即增加了系统的安全风险，又消耗了系统的大量资源。但是机器和人是有区别的，比如人是不可能在非常短的时间内（比如 `100ms` 内）发送多个请求的，或者正常用户不会在一个短时间内，比如 1 个小时尝试 3000 次同一个接口的请求。
* 加固请求来源的信任 -- 对于安全性高的系统会要求客户端（ `iOS` 、 `Android` 等）上传自身应用的签名，避免有人反编译程序或者通过抓包等进行分析，重新伪造请求。对于 Web 应用可以通过 CORS 指定某些地址的请求才可以访问
* 黑名单 -- 这个就是根据各种不正常的用户行为或者已经查明的攻击来源进行拉黑处理

当然还有很多其他的手段，安全是一个大问题，但不是本书的主要目的，所以就不展开讨论了。

## 使用 Spring Security 规划角色安全

Spring Security 是一个基于 Spring 的通用安全框架，里面内容太多了，我们的主要目的也不是展开讲这个框架，而是如何利用 Spring Security 和 JWT 一起来完成 API 保护。所以关于 Spring Secruity 的基础内容或展开内容，请自行去官网学习<http://projects.spring.io/spring-security> 。

如果你的系统有用户的概念的话，一般来说，你应该有一个用户表，最简单的用户表，应该有三列： `Id` ， `Username` 和 `Password` ，类似下表这种

 ID | USERNAME | PASSWORD
---|---|---
 10 | wang | abcdefg

而且不是所有用户都是一种角色，比如网站管理员、供应商、财务等等，这些角色和网站的直接用户需要的权限可能是不一样的。那么我们就需要一个角色表， Spring Security 中的角色的叫 `AUTHORITY` ：

 ID | AUTHORITY
---|---
 10 | USER
 20 | ADMIN

当然我们还需要一个可以将用户和角色关联起来建立映射关系的表。

 USER_ID | AUTHORITY_ID
---|---
 10 | 10
 20 | 20

这是典型的一个关系型数据库的用户角色的设计，由于我们要使用的 MongoDB 是一个文档型数据库，所以让我们重新审视一下这个结构。

这个数据结构的优点在于它避免了数据的冗余，每个表负责自己的数据，通过关联表进行关系的描述，同时也保证的数据的完整性：比如当你修改角色名称后，没有脏数据的产生。

### MongoDB 中如何实现

在 MongoDB 中我们可以将其简化为有两个文档集合 `user` 和 `authority` ，这两个集合各自是独立的，MongoDB 在数据库层面是没有提供用来定义两个集合关系的操作的，那么我们怎么体现这种关系呢？

一般来说，我们可以在 `User` 对象中设置一个属性，这个属性是一个集合，由 `Authority` 的 Id 组成。当然在 `Authority` 中我们也可以有一个集合，由 `User` 的 Id 构成。这样手动维护了这种关系，但是这个集合最好不要太大，否则会导致这个文档的体积过大，不光是查询性能，增、删、改的性能都会下降。那么我们再来想一下，是否有必要在两个对象中都维护这种关系呢？角色相对的数量是比较少，而且应该也不会增长到很大数字，估计到 100 系统维护人员就得发疯了吧。反过来，有同样角色的用户，这个数量可就不好估计了，如果是面向消费者的系统，那么这个数字就得十万、百万、千万...，所以我们应该只在 `User` 中维护一个 `Authority` 的 Id 组成的集合。

看到这里，估计很多同学会吐槽，这是什么数据库啊，还要手动维护关系。但是大家要注意的是，没有十全十美的方案，所有的方案都是要根据具体场景来做取舍和妥协。大家一定还记得大学里学习数据结构的时候，数组适合查找，但插入和删除就很麻烦。而链表在插入和删除时效率很高，但查找就很麻烦。如果存在一个万能数据结构，我们就不用学习多种数据结构了。

如果你的业务系统中有大量多对多关系，而且对数据完整性有要求的话，那最好还是使用关系型数据库。另外其实一个业务中其实可以有多个数据库，互为补充，在适合的场景使用擅长这个方向存储结构。

再回到我们的数据模型上，现在我们先建立 2 个 `collection` ，一个是 `user` ，一个是 `authority`

```json
# User
{
  _id: <id_generated>
  username: 'zhangsan',
  password: 'pass',
  authorityIds: ['10', '20']
}
{
  _id: <id_generated>
  username: 'lisi',
  password: 'pass',
  authorityIds: ['10']
}
# Authority
{
  _id: '10',
  name: 'USER'
}
{
  _id: '20',
  name: 'ADMIN'
}
```

如果我们希望列出角色为 `USER` 的用户，可以通过以下查询得到

```js
db.user.find({
    authorityIds: "10"
})
```

当然在 MongoDB 3.2 以上版本中，增加了类似 `join` 的功能，比如你可以在 `aggregate` 的 `pipeline` 中使用 `$lookup` 去关联两个 `collection` ，为什么说是类似呢，因为实际上 MongoDB 做的动作是关联了两个大文件，做了两层循环，外层是 `db.xxx` 的这个 `xxx collection` ，内层是要关联的那个 `collection` ，一旦发现符合关联条件的数据就丢到外层的数组中。所以如果是两个非常大数据量的 `collection` 做关联，请首先一定要做索引，其次尽可能利用 `$match` 和 `$project` 减小数据量。而且不要因为有了这个类似 `join` 的功能就按照关系型数据库的经验来做，关于 `$lookup` 推荐大家看 MongoDB 官方团队的解释，虽然是个英文版，但讲的非常清晰。 <https://www.mongodb.com/presentations/doing-joins-in-mongodb-best-practices-for-using-lookup>

我们在 MongoDB 中可以根据下面的 json 建立 User 和 Authority

```json
// Authority
{
    "_id" : ObjectId("5ad88524b78ca780560ac3aa"),
    "name" : "USER"
}
{
    "_id" : ObjectId("5ad8852fb78ca780560ac3ab"),
    "name" : "ADMIN"
}
// User
{
    "_id" : ObjectId("5ad8853db78ca780560ac3ac"),
    "username" : "zhangsan",
    "password" : "pass",
    "authorityIds" : [
        ObjectId("5ad88524b78ca780560ac3aa")
    ]
}
{
    "_id" : ObjectId("5ad885c2b78ca780560ac3ad"),
    "username" : "lisi",
    "password" : "pass",
    "authorityIds" : [
        ObjectId("5ad8852fb78ca780560ac3ab"),
        ObjectId("5ad88524b78ca780560ac3aa")
    ]
}
```

然后使用 MongoDB 的 `$lookup` 将所有角色名为 `ADMIN` 的用户列出来。

```js
db.user.aggregate(
    [
        {
            "$unwind" : {
                "path" : "$authorityIds"
            }
        },
        {
            "$lookup" : {
                "from" : "authority",
                "localField" : "authorityIds",
                "foreignField" : "_id",
                "as" : "authorities"
            }
        },
        {
            "$match" : {
                "authorities.name" : "ADMIN"
            }
        }
    ],
    {
        "allowDiskUse" : false
    }
);
```

## 在 SpringBoot 中启用 Spring Security

要在 Spring Boot 中引入 Spring Security 非常简单，修改 `build.gradle`，增加一个引用 `org.springframework.boot:spring-boot-starter-security`：

```groovy
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("io.jsonwebtoken:jjwt:0.9.0")
    implementation("org.springframework.boot:spring-boot-starter-security")
    implementation("org.springframework.boot:spring-boot-starter-aop")
    implementation("org.zalando:problem-spring-web:0.20.1")
    implementation("com.fasterxml.jackson.module:jackson-module-afterburner")
    implementation("org.springframework.boot:spring-boot-starter-data-mongodb")
}
```

## 改造用户对象

为什么需要对用户对象进行改造？其一是我们需要把用户拥有的角色放入 JWT Token 中，传递给客户端，这样客户端可以根据需要决定那些界面功能需要开放给用户还是隐藏起来。其二是 Spring Security 需要我们实现几个东西，第一个是 `UserDetails` ：这个接口中规定了用户的几个必须要有的方法，所以我们要让 `User` 类添加几个属性以满足这个接口。为什么不直接实现 `UserDetails` 类？因为这个 `UserDetails` 完全是为了安全服务的，它和我们的领域类可能有部分属性重叠，但很多的接口其实是为了 Spring Security 的方式定制的，如果以后我们采用别的安全框架的话，这样的紧耦合是不太好的。因此我们需要做一些隔离，但不管采用那个框架，我们自己 User 类确实还需要添加几个属性：

* `activated` -- 标识用户是否激活
* `resetKey` -- 重置密码密钥
* `resetDate` -- 重置密码时间
* `authorities` -- 用户的角色集合

```java
package dev.local.gtm.api.domain;

import com.fasterxml.jackson.annotation.JsonIgnore;
import dev.local.gtm.api.config.Constants;
import lombok.*;
import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.index.Indexed;
import org.springframework.data.mongodb.core.mapping.DBRef;
import org.springframework.data.mongodb.core.mapping.Document;
import org.springframework.data.mongodb.core.mapping.Field;

import javax.validation.constraints.Email;
import javax.validation.constraints.NotNull;
import javax.validation.constraints.Pattern;
import javax.validation.constraints.Size;
import java.io.Serializable;
import java.time.Instant;
import java.util.Set;

@Data
@Builder
@EqualsAndHashCode(callSuper = false, of = {"id"})
@ToString(exclude = "authorities")
@NoArgsConstructor
@AllArgsConstructor
@Document(collection = "api_users")
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

    @Size(max = 20)
    @Field("reset_key")
    @JsonIgnore
    private String resetKey;

    @Field("reset_date") @Builder.Default
    private Instant resetDate = null;

    @Builder.Default
    private boolean activated = false;

    @JsonIgnore
    @Singular("authority")
    @DBRef(lazy = true)
    @Field("authority_ids")
    private Set<Authority> authorities;
}
```

`@Singular("authority")` 这个是和 Lombok 提供的 `@Builder` 配合为 `User` 类增加一个可以单独添加一个角色到集合的方法。而括号中的 `authority` 就是在 `Builder` 中这个方法的名称，一般集合起的属性名都是复数形式（ `authorities` ）， `@Singular("authority")` 就是我们给添加单个元素的方法起个名称，这里就是 `authority` 了。具体的用法如下面代码所示：

```java
User.builder().authority(new Authority("Blablabla")).build();
```

我们在 `User` 的 `authorities` 属性上还应用了一个注解 `@DBRef(lazy = true)` ，之前我们提过在 MongoDB 3.2 以上支持类似关系型数据库的多个 collection 的查询，那么这个 `@DBRef` 就类似于标识“外键”的感觉 <https://docs.mongodb.com/manual/reference/database-references>。

* 手动引用 -- 我们手动保存某个文档的 `_id` 到另一个文档中作为引用。应用可以根据这些引用执行新的查询得到想要的数据，一般情况下，这种方式是完全够用的。
* `DBRef` 是把文档的 `_id` 和 `collection` 名称以及可选的数据库名称一起保存起来。和手动方式类似，这种方式也需要查询两次，很多 MongoDB 的驱动提供了一些工具类，但是这些驱动并不会自动解析 `DBRef` 。可以这么理解， `DBRef` 提供了一种通用的格式来表示文档之间的关系。

值得指出的是 MongoDB 官方并不推荐使用 `DBRef` ，而是推荐尽可能的手动关联。因为通过手动方式有很多情况下你可以避免多次查询，比如除了 `_id` 我们还可以冗余某些字段到文档中，而 `DBRef` 只是保存了 `_id` 。但在这个例子中，角色的数量不会很多，而且我们后面想要提供角色名称的更改功能，所以这里我们还是使用了 `DBRef` 。在实际工作中，请慎重使用 `DBRef` 。

Spring 对于 `DBRef` 也做了一些优化，比如注解中的 `lazy = true` 就是给 DBRef 字段懒加载的特性，只有使用的时候才会去查询，避免无谓的二次查询。

## 构建 JWT token 工具类

那么改造完用户这个领域对象之后，我们需要根据这个用户的信息生成 JWT token，也需要在请求中得到 token 的时候检验这个 token 是否合法，还需要得到这个合法的 token 之后可以解析它。所以呢，我们在 `api` 包下面新建一个 `security` 包，然后在 `security` 下面再建立一个 `jwt` 包，在这个包下面新建 `TokenProvider.java`

```java
package dev.local.gtm.api.security.jwt;

import dev.local.gtm.api.config.AppProperties;
import io.jsonwebtoken.*;
import lombok.RequiredArgsConstructor;
import lombok.extern.log4j.Log4j2;
import lombok.val;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.userdetails.User;
import org.springframework.stereotype.Component;

import javax.annotation.PostConstruct;
import java.util.Arrays;
import java.util.Date;
import java.util.stream.Collectors;

/**
 * Jwt Token 的工具类
 *
 * @author Peng Wang (wpcfan@gmail.com)
 */
@Log4j2
@RequiredArgsConstructor
@Component
public class TokenProvider {

    private static final String AUTHORITIES_KEY = "auth";

    private String secretKey;

    private long tokenValidityInMilliseconds;

    private final AppProperties appProperties;

    @PostConstruct
    public void init() {
        this.secretKey = appProperties.getSecurity().getJwt().getSecret();
        this.tokenValidityInMilliseconds = 1000 * appProperties.getSecurity().getJwt().getTokenValidityInSeconds();
    }

    public String createToken(Authentication authentication) {
        val authorities = authentication.getAuthorities().stream()
                .map(GrantedAuthority::getAuthority)
                .collect(Collectors.joining(","));

        val now = (new Date()).getTime();
        val validity = new Date(now + this.tokenValidityInMilliseconds);

        return Jwts.builder()
                .setSubject(authentication.getName())
                .claim(AUTHORITIES_KEY, authorities)
                .signWith(SignatureAlgorithm.HS512, secretKey)
                .setExpiration(validity)
                .compact();
    }

    public Authentication getAuthentication(String token) {
        val claims = Jwts.parser()
                .setSigningKey(secretKey)
                .parseClaimsJws(token)
                .getBody();

        val authorities =
                Arrays.stream(claims.get(AUTHORITIES_KEY).toString().split(","))
                        .map(SimpleGrantedAuthority::new)
                        .collect(Collectors.toList());

        val principal = new User(claims.getSubject(), "", authorities);

        return new UsernamePasswordAuthenticationToken(principal, token, authorities);
    }

    public boolean validateToken(String authToken) {
        try {
            Jwts.parser().setSigningKey(secretKey).parseClaimsJws(authToken);
            return true;
        } catch (SignatureException e) {
            log.info("非法 JWT 签名");
            log.trace("非法 JWT 签名的 trace: {}", e);
        } catch (MalformedJwtException e) {
            log.info("非法 JWT token.");
            log.trace("非法 JWT token 的 trace: {}", e);
        } catch (ExpiredJwtException e) {
            log.info("过期 JWT token");
            log.trace("过期 JWT token 的 trace: {}", e);
        } catch (UnsupportedJwtException e) {
            log.info("系统不支持的 JWT token");
            log.trace("系统不支持的 JWT token 的 trace: {}", e);
        } catch (IllegalArgumentException e) {
            log.info("JWT token 压缩处理不正确");
            log.trace("JWT token 压缩处理不正确的 trace: {}", e);
        }
        return false;
    }
}
```

这个文件提供了三个方法

* `createToken` -- sub 设为用户的登录名， `auth` 为其拥有的角色（数组形式），当然我们可以再放一些其他信息进去，比如用户的姓名等。
  * token 中的信息如果是客户端常用的信息就可以减少客户端对 API 的请求频次和数量。但不要在 token 中放入敏感的信息，因为即使没有签名也能解开 token 中的其他信息。
  * 在这个方法中我们还设置了过期时间，一般的规则是 token 的生命周期不要太长，比如一般不要给一个几个月或一年都有效的 token，测试目的除外。实际工作中都是设一个几个小时有效的 token。
* `getAuthentication` -- 使用签名解析 JWT token ，取出相关信息形成 Spring Security 中的 `UsernamePasswordAuthenticationToken` ，Spring Security 根据这个去判断用户是否已经鉴权。
  * 值得注意的是，虽然叫做 `UsernamePasswordAuthenticationToken` ，但我们构造的时候使用了 token 作为 password 参数。换句话说如果 token 合法，我们就认为这个用户是已登录的用户。
  * 那么为什么我们方法定义中写返回应该是 `Authentication` ，但实际返回的是 `UsernamePasswordAuthenticationToken` ？因为 `UsernamePasswordAuthenticationToken` 实现了 `Authentication` 接口。
* `validateToken` -- 验证 token 是否合法

大家可能留意到了，我们使用了 `AppProperties` ，而且似乎为它添加了几个属性，是的，我们为应用属性添加了 `app.security.authorization` 和 `app.security.jwt` 。

* `app.security.authorization` -- 用于设置授权相关的属性
  * `app.security.authorization.header` -- 在 Http Request 的头当中写的授权信息的键值，也就是 `Authorization: Bearer xxxx(token)` 中 `Authorization` 位置的这个字符串，默认的标准是 `Authorization` ，但也可以允许设置成别的字符串。
* `app.security.jwt` -- 设置 JWT 相关属性
  * `app.security.jwt.secret` -- 设置加密的密钥，这个千万要保存好，否则就是大门洞开了。
  * `app.security.jwt.tokenValidityInSeconds` -- 设置有效期，默认给出值是 7200 秒，也就是 2 个小时。
  * `app.security.jwt.tokenPrefix` -- 就是 `Authorization: Bearer xxxx(token)` 中 `Bearer` 和空格这个字符串

```java
@Value
@Component
@ConfigurationProperties(prefix = "app")
public class AppProperties {

    // 省略其他部分
    private final Security security = new Security();
    // 省略其他部分
    @Data
    public static class Security {
        private final Jwt jwt = new Jwt();
        private final Authorization authorization = new Authorization();

        @Data
        public static class Authorization {
            private String header = "Authorization";
        }

        @Data
        public static class Jwt {
            private String secret = "myDefaultSecret";
            private long tokenValidityInSeconds = 7200;
            private String tokenPrefix = "Bearer ";
        }
    }
}
```

## 如何检查任何请求的授权信息？

有了工具类，我们接下来怎么办，在所有的 API Controller 中应用这个工具进行检查？显然不行，重复代码也太多了吧，我们还是利用 Filter ，这样所有的请求进来时就都被 Filter 检查和处理了。

```java
package dev.local.gtm.api.security.jwt;

import dev.local.gtm.api.config.AppProperties;
import lombok.RequiredArgsConstructor;
import lombok.val;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.util.StringUtils;
import org.springframework.web.filter.GenericFilterBean;

import javax.servlet.FilterChain;
import javax.servlet.ServletException;
import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;
import javax.servlet.http.HttpServletRequest;
import java.io.IOException;

@RequiredArgsConstructor
public class JWTFilter extends GenericFilterBean {

    private final TokenProvider tokenProvider;
    private final AppProperties appProperties;

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain)
            throws IOException, ServletException {
        val httpServletRequest = (HttpServletRequest) servletRequest;
        val jwt = getToken(httpServletRequest);
        if (StringUtils.hasText(jwt) && this.tokenProvider.validateToken(jwt)) {
            val authentication = this.tokenProvider.getAuthentication(jwt);
            SecurityContextHolder.getContext().setAuthentication(authentication);
        }
        filterChain.doFilter(servletRequest, servletResponse);
    }

    private String getToken(HttpServletRequest request){
        val bearerToken = request.getHeader(appProperties.getSecurity().getAuthorization().getHeader());
        val prefix = appProperties.getSecurity().getJwt().getTokenPrefix();
        if (StringUtils.hasText(bearerToken) && bearerToken.startsWith(prefix)) {
            return bearerToken.substring(prefix.length(), bearerToken.length());
        }
        val jwt = request.getParameter(appProperties.getSecurity().getAuthorization().getHeader());
        if (StringUtils.hasText(jwt)) {
            return jwt;
        }
        return null;
    }
}
```

这个 Filter 做的事情就是当请求进来后，我们先从这个 Request 的头部把授权信息取出来，然后返回解析后的 JWT token 。如果这个 token 是合法的，那么我们就使用工具类得到 Spring Security 中的鉴权 `val authentication = this.tokenProvider.getAuthentication(jwt)` 。最后在 Spring Security 的安全上下文中设置这个鉴权信息 `SecurityContextHolder.getContext().setAuthentication(authentication)` 。

多说两句 `SecurityContextHolder` 是一个关联 `SecurityContext` （安全上下文） 和当前线程的类，一般常用就是使用 `SecurityContextHolder` 的 `getContext()` 得到 `SecurityContext` 。那么这个 `SecurityContext` 又是个什么东东呢？看一下 Spring Security 的源码，哈哈，原来就是存和取 `Authentication` 鉴权对象的啊。

```java
// 摘自 Spring Security 项目，为阅读方便，只保留部分代码
public interface SecurityContext extends Serializable {

    Authentication getAuthentication();

    void setAuthentication(Authentication authentication);
}
```

那么究竟这个 `Authentication` 里面有什么？又可以做到什么呢？还是看源码更清晰

```java
// 摘自 Spring Security 项目，为阅读方便，只保留部分代码
public interface Authentication extends Principal, Serializable {
    // 得到角色集合
    Collection<? extends GrantedAuthority> getAuthorities();

    // 得到密码或者起到密码作用的值，比如 token
    Object getCredentials();

    // 简单理解就是得到用户信息
    Object getDetails();

    // 简单理解就是得到用户名，实际 Principal 的概念要更抽象一些
    Object getPrincipal();

    // 返回是否已鉴权
    boolean isAuthenticated();

    // 设置是否已鉴权
    void setAuthenticated(boolean isAuthenticated) throws IllegalArgumentException;
}
```

这些知识对于更好的理解 Spring Security 是很有帮助的，希望大家可以细致做一些功课，但这里我们就只是略作介绍了。

我们需要回头接着说 Filter ，这个 Filter 建好之后，还得配置到 Spring 中去，否则 Spring 也不知道应该怎么应用这个 Filter 。在 Spring 中 Filter 是有顺序的，而且这个顺序很重要，因为如果你的 Filter 在不适合的位置，那么有可能你的 Filter 本来预期的效果会被其他 Filter 更改掉了。

所以我们在 `jwt` 包中新建一个 `JWTConfigurer.java` ，这个类里面只做了一件事，就是把刚刚的 Filter 加到 Spring Security 内建的 `UsernamePasswordAuthenticationFilter` 之前。也就是说在 Spring Security 内建的鉴权 Filter 开始之前就完成了我们自己的鉴权。

```java
package dev.local.gtm.api.security.jwt;

import dev.local.gtm.api.config.AppProperties;
import lombok.RequiredArgsConstructor;
import lombok.val;
import org.springframework.security.config.annotation.SecurityConfigurerAdapter;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.DefaultSecurityFilterChain;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;
import org.springframework.stereotype.Component;

@RequiredArgsConstructor
@Component
public class JWTConfigurer extends SecurityConfigurerAdapter<DefaultSecurityFilterChain, HttpSecurity> {

    private final TokenProvider tokenProvider;
    private final AppProperties appProperties;

    @Override
    public void configure(HttpSecurity http) {
        val customFilter = new JWTFilter(tokenProvider, appProperties);
        http.addFilterBefore(customFilter, UsernamePasswordAuthenticationFilter.class);
    }
}
```

## 得到用户信息

刚刚我们提到过在 `Authentication` 中可以得到用户信息，这个信息其实是我们自己可以定义的信息，于是接下来要实现的是 `UserDetailsService`，这个接口只定义了一个方法 `loadUserByUsername`，顾名思义，就是提供一种从用户名可以查到用户并返回的方法。注意，不一定是数据库哦，文本文件、xml 文件等等都可能成为数据源，这也是为什么 Spring 提供这样一个接口的原因：保证你可以采用灵活的数据源。接下来我们建立一个 `UserDetailsServiceImpl` 来实现这个接口。

```java
package dev.local.gtm.api.security;

import dev.local.gtm.api.config.Constants;
import dev.local.gtm.api.domain.User;
import dev.local.gtm.api.repository.UserRepo;
import lombok.RequiredArgsConstructor;
import lombok.extern.log4j.Log4j2;
import lombok.val;
import org.hibernate.validator.internal.constraintvalidators.hv.EmailValidator;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.core.userdetails.UsernameNotFoundException;
import org.springframework.stereotype.Component;

import java.util.Locale;
import java.util.regex.Pattern;
import java.util.stream.Collectors;

/**
 * 用户信息服务的具体实现
 *
 * @author Peng Wang (wpcfan@gmail.com)
 */
@Log4j2
@RequiredArgsConstructor
@Component
public class UserDetailsServiceImpl implements UserDetailsService {

    private final UserRepo userRepo;

    /**
     * 通过数据库加载用户信息
     * @param login 用户名
     * @return 返回 Spring Security User
     */
    @Override
    public UserDetails loadUserByUsername(final String login) {
        log.debug("正在对用户名为 {} 的用户进行鉴权", login);

        if (new EmailValidator().isValid(login, null)) {
            val userByEmailFromDatabase = userRepo.findOneByEmailIgnoreCase(login);
            return userByEmailFromDatabase.map(user -> createSpringSecurityUser(login, user))
                    .orElseThrow(() -> new UsernameNotFoundException("系统中不存在 email 为 " + login + " 的用户"));
        }

        if (Pattern.matches(Constants.MOBILE_REGEX, login)) {
            val userByMobileFromDatabase = userRepo.findOneByMobile(login);
            return userByMobileFromDatabase.map(user -> createSpringSecurityUser(login, user))
                    .orElseThrow(() -> new UsernameNotFoundException("系统中不存在手机号为 " + login + " 的用户"));
        }

        String lowercaseLogin = login.toLowerCase(Locale.ENGLISH);
        val userByLoginFromDatabase = userRepo.findOneByLogin(lowercaseLogin);
        return userByLoginFromDatabase.map(user -> createSpringSecurityUser(lowercaseLogin, user))
                .orElseThrow(() -> new UsernameNotFoundException("User " + lowercaseLogin + " was not found in the database"));

    }

    /**
     * 通过应用的用户领域对象创建 Spring Security 的用户
     *
     * 这里有两个 User ，为避免混淆，对于 Spring Security 的 User 采用 Full Qualified Name
     * @param lowercaseLogin 小写的用户登录名
     * @param user 领域对象
     * @return Spring Security User
     * @see org.springframework.security.core.userdetails.User
     */
    private org.springframework.security.core.userdetails.User createSpringSecurityUser(String lowercaseLogin, User user) {
        if (!user.isActivated()) {
            throw new UserNotActivatedException("用户 " + lowercaseLogin + " 没有激活");
        }
        val grantedAuthorities = user.getAuthorities().stream()
                .map(authority -> new SimpleGrantedAuthority(authority.getName()))
                .collect(Collectors.toList());
        return new org.springframework.security.core.userdetails.User(user.getLogin(),
                user.getPassword(),
                grantedAuthorities);
    }
}
```

## 配置 Spring Security

做了这么多前期工作，最后还是得让 Spring Security 我们想如何配置安全，我们还需要建立一个安全配置类 `SecurityConfig` ，这里我们会把 `PasswordEncoder` 和 `AuthenticationManager` 以 `Bean` 的形式提供。在 `void configure(WebSecurity web)` 方法中指定哪些资源是 Spring Security 忽略的，也就是不做鉴权，一般是静态资源。在 `void configure(HttpSecurity http)` 中设置具体的安全策略，包括是否需要鉴权，需要的角色等：

```java
package dev.local.gtm.api.config;

import dev.local.gtm.api.security.AuthoritiesConstants;
import dev.local.gtm.api.security.jwt.JWTConfigurer;
import lombok.RequiredArgsConstructor;
import org.springframework.beans.factory.BeanInitializationException;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Import;
import org.springframework.http.HttpMethod;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.config.annotation.authentication.builders.AuthenticationManagerBuilder;
import org.springframework.security.config.annotation.method.configuration.EnableGlobalMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.builders.WebSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;
import org.zalando.problem.spring.web.advice.security.SecurityProblemSupport;

import javax.annotation.PostConstruct;

@RequiredArgsConstructor
@Configuration
@EnableWebSecurity
@EnableGlobalMethodSecurity(prePostEnabled = true, securedEnabled = true)
@Import(SecurityProblemSupport.class)
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    private final AuthenticationManagerBuilder authenticationManagerBuilder;

    private final UserDetailsService userDetailsService;

    private final SecurityProblemSupport problemSupport;

    private final JWTConfigurer jwtConfigurer;

    @PostConstruct
    public void init() {
        try {
            authenticationManagerBuilder
                    .userDetailsService(userDetailsService)
                    .passwordEncoder(passwordEncoder());
        } catch (Exception e) {
            throw new BeanInitializationException("安全配置失败", e);
        }
    }

    @Override
    @Bean
    public AuthenticationManager authenticationManagerBean() throws Exception {
        return super.authenticationManagerBean();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Override
    public void configure(WebSecurity web) {
        // 以下资源 Spring Security 会忽略，一般用于忽略静态资源
        web.ignoring()
                .antMatchers(HttpMethod.OPTIONS, "/**")
                .antMatchers("/app/**/*.{js,html}")
                .antMatchers("/i18n/**")
                .antMatchers("/content/**")
                .antMatchers("/test/**");
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
                .exceptionHandling()
                .authenticationEntryPoint(problemSupport)
                .accessDeniedHandler(problemSupport)
                .and()
                    .csrf()
                    .disable()
                    .headers()
                    .frameOptions()
                    .disable()
                .and()
                    .sessionManagement()
                    .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
                .and()
                    .authorizeRequests() // 限制 Request 需要认证，以下是不同路径需要的认证方式
                        .antMatchers("/api/auth/**").permitAll() // 无需鉴权
                        .antMatchers("/api/**").authenticated() // 其他路径均需认证
                        .antMatchers("/websocket/tracker").hasAuthority(AuthoritiesConstants.ADMIN) // 不仅需要鉴权而且需要角色为 ADMIN
                        .antMatchers("/management/health").permitAll() // 服务状态接口
                        .antMatchers("/management/**").hasAuthority(AuthoritiesConstants.ADMIN) //服务管理接口
                .and()
                    .apply(jwtConfigurer);
    }
}
```

此时如果我们启动应用，访问 `http://localhost:8080/api/` ，可以看到如下输出，可见我们的安全配置是生效了。

```json
{
    "type":"http://www.twigcodes.com/problem/problem-with-message",
    "title":"Unauthorized",
    "status":401,
    "detail":"Full authentication is required to access this resource",
    "path":"/api/",
    "message":"error.http.401"
}
```

在开始我们的实验前，需要先插入两个角色记录，这是验证鉴权接口的时候我们需要几个现成的角色。这个可以使用 MongoDB 的 shell 或者任一 MongoDB 的图形化管理工具，比如 `Studo 3T` <https://studio3t.com/> 。

```js
db.api_authority.insert({
    "name": "ROLE_USER"
});
db.api_authority.insert({
    "name": "ROLE_ADMIN"
});
```

如果我们使用 Postman 做个实验的话，以 POST 方式访问 <http://localhost:8080/api/auth/register> 并在 `body` 中给出 `json` 的用户注册信息，如下图所示。

![验证注册接口](/assets/2018-04-23-12-15-51.png)

耶！我们成功的得到了 JWT token ，如果我们得到的 token 在 <https://jwt.io> 上进行一个验证的话，可以看到 token 的信息中包含了我们在 `TokenProvider` 中放进去的元素。但是请留意下方红色按钮，上面写着 `Invalid Signature` -- 非法签名。

![在 jwt.io 上验证](/assets/2018-04-23-12-59-33.png)

如果你还记得前面我们在 `AppProperties` 中定义的默认值，这个密钥的默认值是 `myDefaultSecret`

```java
// 省略其他代码
public class AppProperties {
    // 省略其他代码
    private final Security security = new Security();
    // 省略其他代码
    @Data
    public static class Security {
        private final Jwt jwt = new Jwt();
        // 省略其他代码
        @Data
        public static class Jwt {
            private String secret = "myDefaultSecret";
            private long tokenValidityInSeconds = 7200;
            private String tokenPrefix = "Bearer ";
        }
    }
    // 省略其他代码
}
```

所以我们把这个值粘贴到写着 `secret` 那个输入框，你就会发现红色的按钮变成蓝色，文字也变成了 `Signature Verified` -- 签名已验证。好的，这也验证了我们生成的 JWT token 是正确的。

## 使用 JWT 进行 API 访问

由于我们在 `SecurityConfig` 中的设置， `/api/auth` 路径下是不需要鉴权的，如果我们想验证一下鉴权访问的结果，需要新建一个 `TaskResource` ，其基础路径为 `/api/tasks` 。

```java
package dev.local.gtm.api.web.rest;

import dev.local.gtm.api.domain.Task;
import dev.local.gtm.api.repository.TaskRepo;
import dev.local.gtm.api.repository.UserRepo;
import dev.local.gtm.api.web.exception.ResourceNotFoundException;
import lombok.RequiredArgsConstructor;
import lombok.extern.log4j.Log4j2;
import lombok.val;
import org.springframework.data.domain.Pageable;
import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@Log4j2
@RestController
@RequestMapping("/api")
@RequiredArgsConstructor
public class TaskResource {
    private final TaskRepo taskRepo;
    private final UserRepo userRepo;

    @GetMapping("/tasks")
    public List<Task> getAllTasks(Pageable pageable, @RequestParam(required = false) String desc) {
        log.debug("REST 请求 -- 查询所有 Task");
        return desc == null ?
                taskRepo.findAll(pageable).getContent() :
                taskRepo.findByDescLike(pageable, desc).getContent();
    }

    @GetMapping("/tasks/search/findByUserMobile")
    public List<Task> findByUserMobile(Pageable pageable, @RequestParam String mobile) {
        log.debug("REST 请求 -- 查询所有手机号为 {} Task", mobile);
        return taskRepo.findByOwnerMobile(pageable, mobile).getContent();
    }

    @PostMapping("/tasks")
    Task addTask(@RequestBody Task task) {
        log.debug("REST 请求 -- 新增 Task {}", task);
        return userRepo.findById(task.getOwner().getId())
                .map(user -> {
                    task.setOwner(user);
                    return taskRepo.save(task);
                })
                .orElseThrow(() -> new ResourceNotFoundException("Task 中 id 为 " + task.getOwner().getId() + " 的 owner 不存在"));
    }

    @PutMapping("/tasks/{id}")
    public Task updateTask(@PathVariable String id, @RequestBody Task toUpdate) {
        log.debug("REST 请求 -- 更新 id: {} 的 Task {}", id, toUpdate);
        val task = taskRepo.findById(id);
        return task.map(res -> {
            res.setDesc(toUpdate.getDesc());
            res.setCompleted(toUpdate.isCompleted());
            res.setOwner(toUpdate.getOwner());
            return taskRepo.save(res);
        }).orElseThrow(() -> new ResourceNotFoundException("id 为 "+ id + " 的 Task 没有找到"));
    }

    @GetMapping("/tasks/{id}")
    public Task getTask(@PathVariable String id) {
        log.debug("REST 请求 -- 取得 id: {} 的 Task", id);
        val task = taskRepo.findById(id);
        return task.orElseThrow(() -> new ResourceNotFoundException("id 为 "+ id + " 的 Task 没有找到"));
    }

    @DeleteMapping("/tasks/{id}")
    @ResponseStatus(HttpStatus.OK)
    public void deleteTask(@PathVariable String id) {
        log.debug("REST 请求 -- 删除 id 为 {} 的Task", id);
        val task = taskRepo.findById(id);
        task.ifPresent(taskRepo::delete);
    }
}
```

这个 Controller 简单的实现了任务的增、删、改、查，为什么没有实现 Service 隔离，而是直接使用 Repository ？一般情况下如果只是简单增删改查操作，个人是不建议增加 Service 层的，单纯的为这种领域对象增加 Service 接口和实现，除了增加了代码的臃肿，没有什么其他好处。只有当业务逻辑复杂起来之后，我们才需要 Service 的封装。

然后我们就可以在 Postman 中发一个 GET 请求访问 <http://localhost:8080/api/tasks>

![直接访问 /api/tasks](/assets/2018-04-23-13-17-49.png)

我们会得到一个 401 的未授权的返回结果，这验证了访问 API 需要鉴权

```json
{
    "type": "http://www.twigcodes.com/problem/problem-with-message",
    "title": "Unauthorized",
    "status": 401,
    "detail": "Full authentication is required to access this resource",
    "path": "/api/tasks",
    "message": "error.http.401"
}
```

而我们如果把鉴权的 Header 在 Postman 中加进去的话，就可以正常得到返回结果了，注意我们得到是一个 200 响应码，和一个空数组，这很正常，因为我们并未添加新的 Task ，但这个 API 经过鉴权是可以访问的了。

![添加授权头](/assets/2018-04-23-13-25-29.png)

下面的问题就是如果同样是系统用户，但如何限制不同角色的用户可以访问不同的资源？

## 使用角色

接下来我们要规定一下哪些资源需要什么样的角色可以访问了，首先将 `TaskResource` 改造一下，我们给方法加上注解，现在只利用两个注解 `@PreAuthorize` 和 `@PostAuthorize` 。简单说一下下面代码的逻辑

* `getAllTasks` -- 查询任务，默认情况下每个人只能得到自己的任务列表，而管理员可以得到所有的任务列表。这个权限是数据的查询方式不一样，虽然也可以使用注解，比如 `@PostFilter` ，但这种过滤是全部查出来之后过滤，浪费资源。我们这里采用的是根据角色给出不同的查询。
* `findByUserMobile` -- 给管理员一个可以通过手机号过滤任务的方法
* `addTask` -- 每个人创建任务的 owner 只能是自己（这个需求后面有变化，暂时这里先这么做），换句话就是自己建立的任务的所有者就是自己
* `updateTask` -- 每个人只能更新自己的任务，当然管理员除外，管理员可以更新所有人的任务。
* `getTask` -- 取得某一个任务，还是每个人只能取自己创建的任务，但管理员除外。
* `deleteTask` -- 删除任务，这个我们先不加权限，后面再处理。

```java
public class TaskResource {
    private final TaskRepo taskRepo;
    private final UserRepo userRepo;

    @GetMapping("/tasks")
    public Page<Task> getAllTasks(Pageable pageable) {
        log.debug("REST 请求 -- 查询所有 Task");
        return SecurityUtils.isCurrentUserInRole(AuthoritiesConstants.ADMIN) ?
                taskRepo.findAll(pageable) :
                SecurityUtils.getCurrentUserLogin().map(login -> taskRepo.findByOwnerLogin(pageable, login))
                        .orElseThrow(() -> new InternalServerErrorException("未找到当前登录用户的登录名"));
    }

    @PreAuthorize("hasRole('ADMIN')")
    @GetMapping("/tasks/search/findByUserMobile")
    public List<Task> findByUserMobile(Pageable pageable, @RequestParam String mobile) {
        log.debug("REST 请求 -- 查询所有手机号为 {} Task", mobile);
        return taskRepo.findByOwnerMobile(pageable, mobile).getContent();
    }

    @PostMapping("/tasks")
    Task addTask(@RequestBody Task task) {
        log.debug("REST 请求 -- 新增 Task {}", task);
        return SecurityUtils.getCurrentUserLogin()
                .flatMap(userRepo::findOneByLogin)
                .map(user -> {
                    task.setOwner(user);
                    return taskRepo.save(task);
                })
                .orElseThrow(() -> new ResourceNotFoundException("未找到用户鉴权信息"));
    }

    @PreAuthorize("#toUpdate.owner.login == principal.username or hasRole('ADMIN')")
    @PutMapping("/tasks/{id}")
    public Task updateTask(@PathVariable String id, @RequestBody Task toUpdate) {
        log.debug("REST 请求 -- 更新 id: {} 的 Task {}", id, toUpdate);
        val task = taskRepo.findById(id);
        return task.map(res -> {
            res.setDesc(toUpdate.getDesc());
            res.setCompleted(toUpdate.isCompleted());
            val user = userRepo.findById(toUpdate.getOwner().getId());
            if (user.isPresent()) {
                res.setOwner(user.get());
            } else {
                throw new ResourceNotFoundException("id 为 "+ id + " 的 User 没有找到");
            }
            return taskRepo.save(res);
        }).orElseThrow(() -> new ResourceNotFoundException("id 为 "+ id + " 的 Task 没有找到"));
    }

    @PostAuthorize("returnObject.owner.login == principal.username or hasRole('ADMIN')")
    @GetMapping("/tasks/{id}")
    public Task getTask(@PathVariable String id) {
        log.debug("REST 请求 -- 取得 id: {} 的 Task", id);
        val task = taskRepo.findById(id);
        return task.orElseThrow(() -> new ResourceNotFoundException("id 为 "+ id + " 的 Task 没有找到"));
    }

    @DeleteMapping("/tasks/{id}")
    @ResponseStatus(HttpStatus.OK)
    public void deleteTask(@PathVariable String id) {
        log.debug("REST 请求 -- 删除 id 为 {} 的Task", id);
        val task = taskRepo.findById(id);
        task.ifPresent(taskRepo::delete);
    }
}
```

Spring Security 提供 4 个表达“之前”或“之后”意义的注解，这四个注解在我们的 `SecurityConfig` 中以 `@EnableGlobalMethodSecurity(prePostEnabled = true)` 这种形式启用，分别的用途是：

* `@PreAuthorize` -- 在方法调用之前检验其表达式，以确定是否有权调用方法，这个注解也是四个中最常用的一个。
* `@PostAuthorize` -- 在方法执行之后检验其表达式，以决定是否有权得到返回值。一般使用的场景是，在方法使用期无法得到某些数据，执行之后才可以，而权限表达式又需要这个值，此时就可以使用这个注解。但要注意一般如果方法是要改变某些数据的时候，要慎重使用这个注解，因为方法实际上已经执行，无论你有没有权限，限制的只是返回结果而已。通常会使用到内建的变量 `returnObject` -- 方法的返回值。
* `@PreFilter` -- 在方法执行前对数据进行筛选，但使用的场景较少，一般是在得到数据之后进行筛选，而不是之前。
* `@PostFilter` -- 在方法执行后对得到的数据进行筛选。需要注意这个筛选对于大数量级的数据来说是不适合的。

所以 `@PreAuthorize("hasRole('ADMIN')")` 的意思就上班只能有 `ROLE_ADMIN` 角色的用户有权调用， 需注意的一点是 `hasRole` 表达式认为每个角色名字前都有一个前缀 `ROLE_`。所以这里的 `ADMIN` 其实在数据库中存储的是 `ROLE_ADMIN` 。 而 `hasRole` 就是一个内建表达式， Spring Security 提供的常用的内建表达式见下表：

内建表达式 | 功能描述
---|---
`hasRole(role)` | 如果当前用户有该角色是返回 true ，否则 false 。角色名称默认是不加 `ROLE_` 前缀的。
`hasAnyRole([role1,role2])` | 和上面的区别就是这个是只要有多个角色中的一个就返回 true
`principal` | 当前用户的 `Principal` 对象，很多时候我们会在表达式中使用用户名信息 `principal.username` 或激活状态 `principal.enabled` 等
`authentication` | Authentication 对象
`permitAll` | 永远返回 `true`
`denyAll` | 永远返回 `false`
`isAnonymous()` | 如果是未鉴权用户就返回 `true`
`isAuthenticated()` | 是否已鉴权
`hasPermission(Object target, Object permission)` | 如果用户对目标对象有访问权限的话返回 `true`
`hasPermission(Object targetId, String targetType, Object permission)` | 和上面类似，只不过多了一个参数 `id`

这种表达式中如果想引用方法的参数，可以使用 `#` ，比如下面的表达式中的 `#toUpdate` 就是方法的参数 `@RequestBody Task toUpdate` 。

```java
@PreAuthorize("#toUpdate.owner.login == principal.username or hasRole('ADMIN')")
    @PutMapping("/tasks/{id}")
    public Task updateTask(@PathVariable String id, @RequestBody Task toUpdate) {
        // 省略方法
    }
```

前面的 `TaskResource` 的代码中，我们还使用了一个工具类 `SecurityUtils` ，这个工具类比较简单，就是进一步封装了 Spring Security 的一些方法，让我们使用起来更方便。

```java
package dev.local.gtm.api.security;

import org.springframework.security.core.context.SecurityContext;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.core.userdetails.UserDetails;

import java.util.Optional;

/**
 * Spring Security 的工具类
 */
public final class SecurityUtils {

    private SecurityUtils() {
    }

    /**
     * 得到当前用户的登录名
     *
     * @return 返回当前用户的登录名
     */
    public static Optional<String> getCurrentUserLogin() {
        SecurityContext securityContext = SecurityContextHolder.getContext();
        return Optional.ofNullable(securityContext.getAuthentication())
                .map(authentication -> {
                    if (authentication.getPrincipal() instanceof UserDetails) {
                        UserDetails springSecurityUser = (UserDetails) authentication.getPrincipal();
                        return springSecurityUser.getUsername();
                    } else if (authentication.getPrincipal() instanceof String) {
                        return (String) authentication.getPrincipal();
                    }
                    return null;
                });
    }

    /**
     * 得到当前用户的 JWT token
     *
     * @return 返回当前用户的 JWT toekn
     */
    public static Optional<String> getCurrentUserJWT() {
        SecurityContext securityContext = SecurityContextHolder.getContext();
        return Optional.ofNullable(securityContext.getAuthentication())
                .filter(authentication -> authentication.getCredentials() instanceof String)
                .map(authentication -> (String) authentication.getCredentials());
    }

    /**
     * 用户是否已鉴权
     *
     * @return 已鉴权返回 true 否则返回 false
     */
    public static boolean isAuthenticated() {
        SecurityContext securityContext = SecurityContextHolder.getContext();
        return Optional.ofNullable(securityContext.getAuthentication())
                .map(authentication -> authentication.getAuthorities().stream()
                        .noneMatch(grantedAuthority -> grantedAuthority.getAuthority().equals(AuthoritiesConstants.ANONYMOUS)))
                .orElse(false);
    }

    /**
     * 当前用户是否有指定角色
     *
     * @param authority 要检查的角色
     * @return 如果有该角色返回 true 否则返回 false
     */
    public static boolean isCurrentUserInRole(String authority) {
        SecurityContext securityContext = SecurityContextHolder.getContext();
        return Optional.ofNullable(securityContext.getAuthentication())
                .map(authentication -> authentication.getAuthorities().stream()
                        .anyMatch(grantedAuthority -> grantedAuthority.getAuthority().equals(authority)))
                .orElse(false);
    }
}
```

现在大家可以去试着通过 `/api/auth/register` 接口注册几个新用户，其中 `ADMIN` 角色的用户可以在 MongoDB 中手动修改一下，然后使用他们的 `token` 去创建各自的任务，体验一下角色的对于资源的限制。

下面我们来做一个验证，首先创建两个用户 `admin` （ `ROLE_ADMIN` 和 `ROLE_USER` 两个角色） 和 `user2` （ `ROLE_USER` 角色） 。

```js
db.api_users.insert({
    "_id" : ObjectId("5add712330ccb9a9e7f95f7b"),
    "login" : "admin",
    "name" : "admin user",
    "email" : "admin@local.dev",
    "mobile" : "13012340001",
    "activated" : true,
    "authority_ids" : [
        DBRef("api_authority", ObjectId("5add5d2c301b322e28e288f3")),
        DBRef("api_authority", ObjectId("5add5d48301b322e28e288f4"))
    ],
    "password" : "$2a$10$.DN8bpMCQxkyvLjhfq5ht.6v8U6cICyW3aU57nizkBsuPHcYyF.4y",
    "avatar" : "/img/avatar/2.jpg"
});
db.api_users.insert({
    "_id" : ObjectId("5adda1eaf68cb3a9a905dc55"),
    "login" : "user2",
    "password" : "$2a$10$TiGrPs11iL3rqmVclg2mhOIiviXuB6NbkwFRXiCjc2gxaGJBpjuUe",
    "mobile" : "13012340002",
    "name" : "李四",
    "email" : "lisi@local.dev",
    "avatar" : "/img/avatar/3.jpg",
    "activated" : true,
    "authority_ids" : [
        DBRef("api_authority", ObjectId("5add5d2c301b322e28e288f3"))
    ],
    "created_date" : ISODate("2018-04-23T09:05:46.922+0000"),
    "last_modified_date" : ISODate("2018-04-23T09:05:46.922+0000"),
    "_class" : "dev.local.gtm.api.domain.User"
});
```

然后可以利用 `/api/login` 得到每个用户的 token 。分别使用几个用户的 token 创建任务，然后试一下各个接口的权限。

TODO：需要再细化一下，这个地方最好有每种情况的截图和说明

### 安全日志

很多时候，如果 Spring Security 出现问题时，我们希望可以看到日志来帮助我们分析问题，日志怎么开启呢？只需要在 `application.yml` 中配置 `org.springframework.security: DEBUG` 即可。

```yml
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
```

其他日志也可以采用这种方式开启，比如上面我们设置了 `org.springframework.web.client.RestTemplate: DEBUG` ，这个是开启 RestTemplate 的请求日志，可以看到每个对外请求的细节信息。

而 `org.springframework.data.mongodb.core.MongoTemplate: DEBUG` 可以看到组装的 MongoDB 的查询语句。

### 多环境的配置

上面在做各种实验时，我不知道大家是否留意到了，我们操作的其实是真正的数据库，在平时做一些个人的探索项目时倒也无所谓。但如果在商业项目中，如果我们直接去访问生产库是绝对不允许的。当然除了数据库，比如有很多第三方 API ，或者其他的配置，在生产环境和开发环境中也是很不一样的。这就引出了一个现代开发中的重要概念 -- 多环境配置。

在 Spring Boot 中对于多环境配置有良好的支持，拿 `yml` 配置文件来说，我们有两种配置方式：

* 单文件形式：在 `application.yml` 中以 `---` 区隔不同的环境配置。
* 多文件形式：以 `applicaiton-环境配置名.yml` 形式提供多个文件。

```yml
# 单文件形式
spring:
  application:
    name: api-service
  profiles:
    active: dev
  data:
    mongodb:
      database: gtm-api
server:
  port: 8080
logging:
  level:
    org.apache.http: ERROR
    org.springframework:
      web: ERROR
      data: ERROR
      security: ERROR
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
  profiles: test
  data:
    mongodb:
      database: gtm-api-test
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

如果配置项不是很多的时候，采用单文件的就可以了， `yml` 支持环境配置的“继承”，比如文件开始的那段就是公共配置，每个环境只需要改动在自己这里发生变化的即可，不写的自动使用公共配置。大家可以看到我们对于不同环境使用了不同的数据库和不同的日志。具体的环境配置名称使用类似 `profiles: test` 形式进行命名。

#### 使用不同的环境

在 IDEA 中，我们可以在 `Run/Debug Configurations` 中设置 `Active Profiles`，比如填 `dev` 、 `test` 或 `prod` 等

![IDEA 中设置 profile](/assets/2018-04-23-21-12-41.png)

如果使用 `gradle` 命令行的话，需要在 `api/build.gradle` 中增加 `bootRun` 区块

```groovy
apply plugin: 'org.springframework.boot'
repositories {
    maven { setUrl('http://oss.jfrog.org/artifactory/oss-snapshot-local/') }
}
configurations {
    // 省略
}
bootRun {
    systemProperties = System.properties as Map<String, ?>
}
test {
    systemProperties['spring.profiles.active'] = 'test'
}
dependencies {
// 省略
}
```

然后就可以使用下面的命令运行 `prod` 环境，当然你可以试试其他的环境配置，并在不同环境配置下使用 API 创建任务或注册用户，可以看看 MongoDB 中是不是不同的配置下数据库是不一样的。

```bash
./gradlew :api:bootRun -Dspring.profiles.active=prod
```
