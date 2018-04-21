# Controller 的构建

## 改造 TaskRepo 和 UserRepo

`TaskRepo` 目前先保持前面改造后的样子。

```java
@Repository
public interface TaskRepo extends MongoRepository<Task, String> {
    Page<Task> findByDescLike(Pageable pageable, @Param("desc") String desc);
    Page<Task> findByOwnerMobile(Pageable pageable, @Param("mobile") String mobile);
}
```

`UserRepo` 需要在用户注册时确保用户名、电子邮件和手机号唯一，因此需要检查这几项是否在数据库中已经存在。对于这几项检查是每个用户注册时都需要做的，而且由于可能会出现同一用户多次提交，比如用户名提示已存在，更改后还是已存在，所以我们给这几个查询分别做缓存 `@Cacheable(cacheNames = USERS_BY_XXX_CACHE)` 。

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

这里面还需要指出的是，对于 Java 8 的 `Optional` 不熟悉的同学，可以去看 Oracle 的一个官方教程 <http://www.oracle.com/technetwork/articles/java/java8-optional-2175753.html> 。

## 实现 Controller

实现用户的登录、注册和忘记密码，我们接下来逐一分析它们的业务逻辑

* 登录 -- 需要传入用户的用户名和密码，然后根据用户名查找用户，如果系统中不存在该用户，则需要以异常形式告诉客户端。如果存在，则继续比较密码是否相同，如果相同则以 `json` 形式返回该用户信息，否则需要以异常形式提示用户名密码不匹配。
* 注册 -- 由于用户的用户名、电子邮件和手机号都应该是唯一的，所以在注册的时候，要先去检查是否这些在系统中已经存在，如果存在需要以异常形式告知用户，没有异常的话就可以新建用户并返回用户信息了。
* 忘记密码 -- 忘记密码其实是一个两个步骤的操作：验证手机和重置密码。所以其实我们这个功能会有两个 API
  * 验证手机： 这里会采用一个第三方的 API （<http://leancloud.cn>） 来去做，也顺便学习一下如何在后端访问第三方的 Rest API。验证的过程是先看手机号是否存在，如果不存在需要以异常形式告知用户。如果存在，将手机号和短信验证码提交给 LeanCloud 的 Rest API 检查，返回的结果如果是成功的，那么就随机生成一个重置密钥保存到 `User` 的 `resetKey` 属性中。如果不成功，以异常形式告知用户。
  * 重置密码：客户端发送手机号、密码和重置密钥到此 API。首先检查是否存在手机号对应的用户，如果不存在，以异常形式告知用户；如果存在，检查该用户的 `resetKey` 是否和传入的密钥匹配，如果不匹配，以异常形式告知用户；如果匹配，按照传入的密码重置该用户密码，且将 `resetKey` 设置为 null。

```java
package dev.local.gtm.api.web.rest;

import dev.local.gtm.api.config.AppProperties;
import dev.local.gtm.api.domain.*;
import dev.local.gtm.api.repository.UserRepo;
import dev.local.gtm.api.util.CredentialUtil;
import dev.local.gtm.api.web.exception.*;
import io.swagger.annotations.ApiOperation;
import lombok.RequiredArgsConstructor;
import lombok.extern.log4j.Log4j2;
import lombok.val;
import org.springframework.http.HttpEntity;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.client.HttpStatusCodeException;
import org.springframework.web.client.RestClientException;
import org.springframework.web.client.RestTemplate;

import java.util.HashMap;

/**
 * 用户鉴权资源接口
 *
 * @author Peng Wang (wpcfan@gmail.com)
 */
@Log4j2
@RestController
@RequestMapping("/api")
@RequiredArgsConstructor
public class AuthResource {

    private final UserRepo userRepo;
    private final RestTemplate restTemplate;
    private final AppProperties appProperties;

    @ApiOperation(value = "用户登录鉴权接口",
            notes = "客户端在 RequestBody 中以 json 传入用户名、密码，如果成功以 json 形式返回该用户信息")
    @PostMapping(value = "/auth/login")
    public User login(@RequestBody final Auth auth) {
        log.debug("REST 请求 -- 将对用户: {} 执行登录鉴权", auth);
        return userRepo.findOneByLogin(auth.getLogin())
                .map(user -> {
                    if (user.getPassword().equals(auth.getPassword())) {
                        throw new InvalidPasswordException();
                    }
                    return user;
                })
                .orElseThrow(LoginNotFoundException::new);
    }

    @PostMapping("/auth/register")
    public ResponseEntity<User> register(@RequestBody User user) {
        log.debug("REST 请求 -- 注册用户: {} ", user);
        if (userRepo.findOneByLogin(user.getLogin()).isPresent()) {
            throw new LoginExistedException();
        }
        if (userRepo.findOneByMobile(user.getMobile()).isPresent()) {
            throw new MobileExistedException();
        }
        if (userRepo.findOneByEmail(user.getEmail()).isPresent()) {
            throw new EmailExistedException();
        }
        return ResponseEntity.ok(userRepo.save(user));
    }

    @PostMapping(value = "/auth/mobile")
    @ResponseStatus(value = HttpStatus.OK)
    public String verifyMobile(@RequestBody MobileVerification verification) {
        log.debug("REST 请求 -- 验证手机号 {} 和短信验证码 {}", verification.getMobile(), verification.getCode());
        return userRepo.findOneByMobile(verification.getMobile())
                .map(user -> {
                    val code = verifySmsCode(verification);
                    if (code.value() != 200) {
                        throw new MobileVerificationFailedException(code.getReasonPhrase());
                    }
                    user.setResetKey(CredentialUtil.generateResetKey());
                    userRepo.save(user);
                    return "{\"resetKey\": \"" + user.getResetKey() + "\"}";
                })
                .orElseThrow(MobileNotFoundException::new);
    }

    @PostMapping(value = "/auth/reset")
    public void resetPassword(@RequestBody KeyAndPassword keyAndPassword) {
        log.debug("REST 请求 -- 重置密码 {}", keyAndPassword);
        userRepo.findOneByMobile(keyAndPassword.getMobile())
                .map(user -> {
                    if (!user.getResetKey().equals(keyAndPassword.getResetKey())) {
                        throw new ResetKeyNotMatchException();
                    }
                    user.setPassword(keyAndPassword.getPassword());
                    user.setResetKey(null);
                    return userRepo.save(user);
                })
                .orElseThrow(LoginNotFoundException::new);
    }

    @PostMapping("/auth/captcha")
    public Captcha verifyCaptcha(@RequestBody final Captcha captcha) {
        val body = new HashMap<String, String>();
        body.put("captcha_code", captcha.getCode());
        body.put("captcha_token", captcha.getToken());
        val entity = new HttpEntity<>(body);
        try {
            val validateCaptcha = restTemplate.postForObject(appProperties.getCaptcha().getVerificationUrl(), entity, Captcha.class);
            if (validateCaptcha == null) {
                throw new InternalServerErrorException("返回对象为空，无法进行验证");
            }
            return Captcha.builder()
                    .code(captcha.getCode())
                    .token(captcha.getToken())
                    .validatedMsg(validateCaptcha.getValidatedMsg())
                    .build();
        } catch (HttpStatusCodeException ex) {
            throw new CaptchaVerificationFailedException(ex.getStatusCode().getReasonPhrase());
        }
    }

    private HttpStatus verifySmsCode(final MobileVerification verification) {
        val body = new HashMap<String, String>();
        body.put("mobilePhoneNumber", verification.getMobile());
        val entity = new HttpEntity<>(body);
        try {
            val response = restTemplate.postForEntity(
                    appProperties.getSmsCode().getVerificationUrl()+"/"+verification.getCode(),
                    entity, Void.class);
            return response.getStatusCode();
        } catch (HttpStatusCodeException ex) {
            return ex.getStatusCode();
        } catch (RestClientException ex) {
            return HttpStatus.INTERNAL_SERVER_ERROR;
        }
    }
}
```

上面的代码中需要讲解的比较多，我们分成几部分来看。

## 登录

登录对应的 API 路径是 `/api/auth/login` ，我们采用 `Post` 方式进行。 `userRepo.findOneByLogin` 返回的是一个 `Optional<User>` ，也就是可能为空， `Optional` 这个 Java 8 带来的新特性很有一些函数式编程的感觉（包括 Java 8 提供的 Stream 也是这样）。和传统的编程方式不太一样，需要一些思维上的转换，一个函数基本要做的事情很简单，但需要转换的思路时要把函数串起来，也就是前一个函数的输出作为后一个的输入。 下面代码中 `...map...orElseThrow` 就是这种感觉。 `map` 中的 `user -> {...}` 是一个 lamda 表达式，也就是一个函数。 `orElseThrow` 中的 `LoginNotFoundException::new` 是一个语法糖，相当于 `_ -> { return new MobileNotFoundException();}` ， 但在只有一个语句时可以简写成 `LoginNotFoundException::new` 。这里面还有一个自定义的异常 `InvalidPasswordException` ，我们放到后面统一讲异常处理，这里先跳过。

```java
@PostMapping(value = "/auth/login")
public User login(@RequestBody final Auth auth) {
    log.debug("REST 请求 -- 将对用户: {} 执行登录鉴权", auth);
    return userRepo.findOneByLogin(auth.getLogin())
            .map(user -> {
                if (user.getPassword().equals(auth.getPassword())) {
                    throw new InvalidPasswordException();
                }
                return user;
            })
            .orElseThrow(LoginNotFoundException::new);
}
```

## 注册

注册对应的 API 路径是 `/api/auth/register` ，同样采用 `Post` 方式进行。注意到 `login` 方法返回的是一个领域对象类型 `User` ，但 `register` 方法返回的却是 `ResponseEntity<User>` ，那么这个 `ResponseEntity` 和前面的 `User` 有什么区别呢？`ResponseEntity` 是完整的一个 Response ，它可以让你控制返回的状态码，返回的对象形式的等等，比如 `ResponseEntity.ok(userRepo.save(user))` 相当于 `ResponseEntity.ok().body(userRepo.save(user))` 。所以如果你确定有要对返回的结果做不同于 Spring 默认的操作的话，可以使用 `ResponseEntity` 。

```java
@PostMapping("/auth/register")
public ResponseEntity<User> register(@RequestBody User user) {
    log.debug("REST 请求 -- 注册用户: {} ", user);
    if (userRepo.findOneByLogin(user.getLogin()).isPresent()) {
        throw new LoginExistedException();
    }
    if (userRepo.findOneByMobile(user.getMobile()).isPresent()) {
        throw new MobileExistedException();
    }
    if (userRepo.findOneByEmail(user.getEmail()).isPresent()) {
        throw new EmailExistedException();
    }
    return ResponseEntity.ok(userRepo.save(user));
}
```

## 忘记密码第一步验证手机

这个方法前多了一个注解 `@ResponseStatus(value = HttpStatus.OK)` 用来以注解形式提供返回的状态码，但如果是 `HttpStatus.OK` 这种，不加也一样，因为默认就是 `OK` ，所以在你不想返回 200 OK 时，又不想使用 `ResponseEntity` 的情况下，可以这样注解达成目的。

另外一个有意思的点是，由于 Java 不是动态语言，所以不可能构建一个临时对象，但是很多时候我们的 API 返回的就是一个简单的字符串，比如

```json
{
    "message": "hello"
}
```

这种情况下为一个简单的字符串构建一个对象实在有点过于繁琐了，所以我们在下面的代码中 `return "{\"resetKey\": \"" + user.getResetKey() + "\"}";` 返回了一个 `json` 字符串，对的，json 其实本身就是字符串，这种方式对待简单的 json 返回是比较方便的，不用重新封装对象。但是也要注意，不能过度使用这种方式。

```java
@PostMapping(value = "/auth/mobile")
@ResponseStatus(value = HttpStatus.OK)
public String verifyMobile(@RequestBody MobileVerification verification) {
    log.debug("REST 请求 -- 验证手机号 {} 和短信验证码 {}", verification.getMobile(), verification.getCode());
    return userRepo.findOneByMobile(verification.getMobile())
            .map(user -> {
                val code = verifySmsCode(verification);
                if (code.value() != 200) {
                    throw new MobileVerificationFailedException(code.getReasonPhrase());
                }
                user.setResetKey(CredentialUtil.generateResetKey());
                userRepo.save(user);
                return "{\"resetKey\": \"" + user.getResetKey() + "\"}";
            })
            .orElseThrow(MobileNotFoundException::new);
}
```

注意在判断验证码的时候，我们使用了一个函数 `verifySmsCode(verification)` ，这个可不是内建函数，是我们自己封装的。这个函数虽然没几行，但是为它准备的周边内容可是不少呢。

```java
// 还记得这个成员变量码？忘了的话翻到前看看代码
private final RestTemplate restTemplate;
// ... 省略其他
private HttpStatus verifySmsCode(final MobileVerification verification) {
    val body = new HashMap<String, String>();
    body.put("mobilePhoneNumber", verification.getMobile());
    val entity = new HttpEntity<>(body);
    try {
        val response = restTemplate.postForEntity(
                appProperties.getSmsCode().getVerificationUrl()+"/"+verification.getCode(),
                entity, Void.class);
        return response.getStatusCode();
    } catch (HttpStatusCodeException ex) {
        return ex.getStatusCode();
    } catch (RestClientException ex) {
        return HttpStatus.INTERNAL_SERVER_ERROR;
    }
}
```

首先来认识一位新朋友 `RestTemplate` 。我们之前都是提供 API 给客户端，但很多时候我们需要自己充当第三方 API 的客户端，这个时候我们就需要使用 `RestTemplate` 了。那么有的同学会想为什么不把这些工作交给前端，它们本来就是纯客户端啊。交给前端会有几个问题

1. 安全性问题 -- 一般 API 都是有密钥的，而且访问时应该传递这个密钥，如果都放在前端，一旦泄漏就是全部的 key 都泄漏了。但在后端就安全很多，因为前端相对脆弱，后端可以通过操作系统和网络的各种安全手段保证被攻破的可能性降低。
2. API 地址过多 -- 一般来说客户端只需要面对自己的后端，当然这也不绝对，有的时候客户端也会使用第三方，这个主要看内部的协调。但 API 接口的地址不宜过多是一个基本原则。
3. 影响范围 -- 如果交给前端管理，那么我们日后要替换某个服务时，就需要从前端到后端全面更改。但如果后端处理，我们可以很大程度上保证接口的兼容性。

### 使用 RestTemplate 发送外部请求

那么书接上回，这个 `RestTemplate` 是怎么得到的呢？我们并没有 `new` 出一个对象来啊。问得好，我们其实是利用了 Spring 的依赖注入特性将这个 `RestTemplate` 以单件构造出来提供给应用使用。所以我们将这个 `restTemplate` 声明成 `private final` 也就是必须的参数，只能通过构造提供初始化。然后使用 `@RequiredArgsConstructor` 提供这样一个构造，这样在 `AuthResource` 中就得到了实例。

接下来我们看看是如何提供这个 `RestTemplate` 实例的，我们在 `api` 包下新建一个 `package` 叫做 config，这个包以后作为我们所有的 Java 配置文件的位置。在下面新建一个 `OutboundRestTemplateConfig.java` 。

```java
package dev.local.gtm.api.config;

import dev.local.gtm.api.interceptor.LeanCloudRequestInterceptor;
import lombok.RequiredArgsConstructor;
import org.springframework.boot.web.client.RestTemplateBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.client.RestTemplate;

/**
 * 为应用访问外部 Http Rest API 提供配置
 *
 * @author Peng Wang (wpcfan@gmail.com)
 */
@RequiredArgsConstructor
@Configuration
public class OutgoingRestTemplateConfig {

    private static final int TIMEOUT = 5000;
    private final AppProperties appProperties;

    @Bean
    public RestTemplate getRestTemplate() {
        return new RestTemplateBuilder()
                .setConnectTimeout(TIMEOUT)
                .interceptors(new LeanCloudRequestInterceptor(appProperties))
                .build();
    }
}
```

这个文件非常简单，我们利用 `RestTemplateBuilder` 构造一个 `RestTemplate` ，并加上了 `@Bean` 注解，这样只要你在应用内声明 `RestTemplate` 类型的成员变量，并提供了构造函数以供注入，系统就会找到这个实例注入进去。这样提供的好处在于我们不需要每次都 new 一个 `RestTemplate` 了，避免了一些消耗，同时为 `RestTemplate` 设置了一些通用的属性，避免团队合作时，每个人使用的参数不统一。

### 实现 ClientHttpRequestInterceptor 进行外部请求拦截

注意在构造 `RestTemplate` 的时候，我们设置了超时的时间和一个请求拦截器。和 Spring MVC 的拦截器概念类似，只不过 Spring MVC 中的拦截器是拦截进来的请求 ( `incoming` )，而这里我们要拦截的是出去的 `outgoing` 。那么为什么要提供拦截器呢？因为第三方 API 服务往往要求 HTTP Request 的鉴权，一般是在 HTTP Request 的头设置一些鉴权需要的字段。而这个如果交给使用者自己添加，一是重复代码太多，二是一旦第三方更改了接口鉴权方式，代码需要改动时，会影响多个位置。所以我们使用一个拦截器帮我们统一管理。

那么我们看看如何实现，在 `api` 包下面新建一个 `interceptor` 的包，然后在此包下面新建一个 `LeanCloudRequestInterceptor.java` 。

```java
package dev.local.gtm.api.interceptor;

import dev.local.gtm.api.config.AppProperties;
import lombok.RequiredArgsConstructor;
import lombok.val;
import org.springframework.http.HttpRequest;
import org.springframework.http.MediaType;
import org.springframework.http.client.ClientHttpRequestExecution;
import org.springframework.http.client.ClientHttpRequestInterceptor;
import org.springframework.http.client.ClientHttpResponse;

import java.io.IOException;

/**
 * 为 LeanCloud 云服务配置在 Request Header 中写入鉴权信息
 * 关于 LeanCloud 短信服务的鉴权信息可以参考 <a>https://leancloud.cn/docs/rest_sms_api.html</a>
 *
 * @author Peng Wang (wpcfan@gmail.com)
 */
@RequiredArgsConstructor
public class LeanCloudRequestInterceptor implements ClientHttpRequestInterceptor {

    private final AppProperties appProperties;

    @Override
    public ClientHttpResponse intercept(HttpRequest request, byte[] body, ClientHttpRequestExecution execution) throws IOException {
        if (!request.getURI().getHost().contains("api.lncld.net")){
            return execution.execute(request, body);
        }
        val headers = request.getHeaders();
        headers.add("X-LC-Id", appProperties.getLeanCloud().getAppId());
        headers.add("X-LC-Key", appProperties.getLeanCloud().getAppKey());
        headers.setContentType(MediaType.APPLICATION_JSON);
        return execution.execute(request, body);
    }
}
```

这个拦截器中我们我们拦截了向外发送的请求，并检查是否是发往 LeanCloud 的请求（ `request.getURI().getHost().contains("api.lncld.net")` ）。如果是的话，就写入 LeanCloud 要求的鉴权头 `X-LC-Id` 和 `X-LC-Key` ，这部分详细信息可以参考 LeanCloud 的官方文档 <https://leancloud.cn/docs/rest_sms_api.html> 。这样处理后，以后再访问 LeanCloud 时就无需写鉴权信息了，让开发者更聚焦在业务层面。

你可能关注到了，我们漏了一个地方没有讲，这个 `LeanCloudProperties` 是什么鬼？

### 为应用提供外部可配置属性的能力

前面我们提到过在 `application.yml` 或者 `application.properties` 中可以配置很多属性，但你有没有好奇过我们是否可以添加自己属性呢？这个 Spring 是支持的，而且非常之简单，只需要在一个类注解 `@ConfigurationProperties("一级属性.二级属性...")` 就行了。

```java
package dev.local.gtm.api.config.propsupport;

import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

/**
 * 为应用配置 LeanCloud 外部属性
 *
 * @author Peng Wang (wpcfan@gmail.com)
 */
@Component
@ConfigurationProperties("app.leancloud")
@Data
public class LeanCloudProperties {
    // 请不要拷贝此处的 appId 和 appKey，这里的是不正确的，请自行申请账号获得自己的 appId 和 appKey
    private String appId = "pqmXXXXXXXXXXXXXXXX-yyyyyy";
    private String appKey = "EUXXXXXXXXXXXXXX";
}
```

那么我们为 LeanCloud 配置了两个鉴权属性 `appId` 和 `appKey` ，这里要记得赋初始值，因为如果在 `.properties` 或 `.yml` 没有设置的时候，程序也得能运行啊。

当然这样写完之后，我们需要在根项目的 `build.gradle` 中添加一个依赖 `spring-boot-configuration-processor` ，它可以对我们的属性注解 `@ConfigurationProperties` 进行处理。

```groovy
// 省略
buildscript {
    // 省略
}

allprojects {
    // 省略
}

subprojects {
    // 省略
    dependencies {
        optional("org.springframework.boot:spring-boot-devtools")
        optional("org.projectlombok:lombok")
        optional("org.springframework.boot:spring-boot-configuration-processor") // <--这里
        testImplementation("org.springframework.boot:spring-boot-starter-test")
    }
    compileJava.dependsOn(processResources)
}
```

现在在 Intellij IDEA 中试试看，在 `application.yml` 中敲 `app.` 的时候还有智能提示呢。

![自定义属性的智能提示](/assets/2018-04-19-11-51-20.png)

但是这么一个个的添加感觉有点乱，我们干脆为整个应用建立一个自己的完整的外部属性配置，在 `config` 中新建 `AppProperties.java` ，这样我们就把所有需要配置的统一到这个 `AppProperties` 中管理了。

```java
package dev.local.gtm.api.config;

import lombok.Data;
import lombok.Value;
import org.springframework.boot.context.properties.ConfigurationProperties;

/**
 * 为本应用服务提供外部可配置的属性支持
 *
 * @author Peng Wang (wpcfan@gmail.com)
 */
@Value
@Component
@ConfigurationProperties(prefix = "app")
public class AppProperties {

    private final LeanCloud leanCloud = new LeanCloud();
    private final SmsCaptcha captcha = new SmsCaptcha();
    private final SmsCode smsCode = new SmsCode();
    private final Security security = new Security();

    @Data
    public static class LeanCloud {
        // 请注册 LeanCloud 获得自己的应用访问的 appID 和 appKey
        private String appId = "blablabla";
        private String appKey = "nahnahnah";
    }

    @Data
    public static class SmsCaptcha {
        // 请注册 LeanCloud 获得自己的应用访问的 URL
        private String requestUrl = "https://xxx.api.lncld.net/1.1/requestCaptcha";
        private String verificationUrl = "https://xxx.api.lncld.net/1.1/verifyCaptcha";
    }

    @Data
    public static class SmsCode {
        // 请注册 LeanCloud 获得自己的应用访问的 URL
        private String requestUrl = "https://xxx.api.lncld.net/1.1/requestSmsCode";
        private String verificationUrl = "https://xxx.api.lncld.net/1.1/verifySmsCode";
    }

    @Data
    public static class Security {
        private Jwt jwt;
        @Data
        public static class Jwt {
            private String secret = "myDefaultSecret";
            private long tokenValidityInSeconds = 7200;
        }
    }
}
```

对 Spring 熟悉的同学可能还知道另一种属性配置的方式，就是 `@Value` 注解了，这两种方式的区别可以看下表。

特性 | `@ConfigurationProperties` | `@Value`
---|---|---
宽松绑定 | 是 | 否
元数据支持 | 是 | 否
SpEL 表达式支持 | 否 | 是

其中宽松绑定是我翻译的名字，英文是 Relaxed Binding <https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-external-config.html#boot-features-external-config-relaxed-binding> ，其实就是不严格要求属性的名称和定义的严格一致，比如 `context-path` 可以绑定到 `contextPath` ，`port` 可以绑定到 `PORT` 等，而 `@Value` 是不支持这个宽松绑定的，因此在环境设置的时候，推荐使用 `@ConfigurationProperties` 。此外 `@Value` 也不支持 IDE 的智能提示，因为没有元数据支持。但在有表达式需求时使用 `@Value` 。

你看，围绕这个 `verifyMobile` 我们讲了这么多，但是这些都是为了更灵活、更方便的开发以后的模块而不得不做先期工作。

## 忘记密码第二步重置密码

讲完了第一步，第二步就简单一些。这里面需要指出的是 `ResetKey` ，因为我们的忘记密码是两个操作，现在我们提供两个接口，但存在一个问题，如果用户直接访问第二个接口怎么办？所以我们需要有一个东西可以关联第一步和第二步。怎么做呢？具体来说就是第一步验证成功后生成一个随机数，存储到 `User` 对象中，第二步客户端要将这个随机数和要修改的密码作为参数传递给第二个接口 `/auth/reset`

```java
@PostMapping(value = "/auth/mobile")
@ResponseStatus(value = HttpStatus.OK)
public String verifyMobile(@RequestBody MobileVerification verification) {
    // 第一步
    return userRepo.findOneByMobile(verification.getMobile())
            .map(user -> {
                // 省略部分代码
                user.setResetKey(CredentialUtil.generateResetKey()); // <-- 这里
                // 省略部分代码
            })
            .orElseThrow(MobileNotFoundException::new);
}

@PostMapping("/auth/reset")
public void resetPassword(@RequestBody KeyAndPassword keyAndPassword) {
    // 第二步
    log.debug("REST 请求 -- 重置密码 {}", keyAndPassword);
    userRepo.findOneByMobile(keyAndPassword.getMobile())
            .map(user -> {
                if (!user.getResetKey().equals(keyAndPassword.getResetKey())) { // <-- 这里
                    throw new ResetKeyNotMatchException();
                }
                user.setPassword(keyAndPassword.getPassword());
                user.setResetKey(null);
                return userRepo.save(user);
            })
            .orElseThrow(LoginNotFoundException::new);
}
```

这个随机数的生成我们做成了一个工具类，我们会放到一个工具类的包中，在 api 下面新建一个 `util` 包，然后在此包下新建一个 `CredentialUtil.java` 。这个工具主要生成一个固定长度的随机数，默认是 10 位。

```java
package dev.local.gtm.api.util;

import lombok.extern.log4j.Log4j2;

import java.util.Random;

@Log4j2
public class CredentialUtil {
    private static final int COUNT = 10;

    /**
     * 生成一个重置密码的随机数，作为激活密钥
     *
     * @return 生成的密钥
     */
    public static String generateActivationKey() {
        return randomNumeric();
    }

    /**
     * 生成一个重置密码的随机数，作为验证密钥
     *
     * @return 生成的密钥
     */
    public static String generateResetKey() {
        return randomNumeric();
    }

    private static String randomNumeric() {
        return String.valueOf(new Random()
                .nextInt((9 * (int) Math.pow(10, CredentialUtil.COUNT - 1)) - 1)
                + (int) Math.pow(10, CredentialUtil.COUNT - 1));
    }
}
```

## API 的异常处理

异常怎么办，上面抛出那么多，也不解释一下吗？嗯，这个异常属于非常重要的一块，所以我们集中在这里讲。

对于一个 API 来说，正常的返回是比较容易定义的，该是个对象就是个对象，该是个数组就是个数组。但是对于异常情况怎么处理是一个比较头疼的问题，如果我们不做任何处理的话，默认出现异常都是返回 500 服务器内部错误。但这个错误的表现形式。一是非常难以理解到底发生了什么，二是对用户非常不友好。

![讨厌的 500 服务器内部错误](/assets/2018-04-19-14-17-29.png)

Spring Boot 提供了一个默认的映射：`/error` ，当处理中抛出异常之后，会转到该请求中处理，并且该请求有一个全局的错误页面用来展示异常内容。我们当然可以定制化这个界面让它更好看一些，包括信息更明确一些。但是我们要构建的是一个前后端分离的应用，所以我们更希望让后端可以告知到底发生了什么，而由前端来处理异常。

![Spring Boot 提供的异常统一处理页面](/assets/2018-04-19-14-22-27.png)

Spring 提供了 `@ControllerAdvice` 以 AOP 的方式进行全局的异常管理。但是如何定一个错误的对象，都需要什么信息？与其自己定义，不如利用已有的、成熟的问题描述方式。这里我们使用一个开源的项目 `problem` <https://zalando.github.io/problem> 来完成我们的问题定义和异常处理。

### 问题的定义

`problem` 把错误和异常统一了名称就叫"问题" `Problem` ，一个 `Problem` 会有几个属性

* 类型 `type`  -- 这个 `type` 是一个 URI 形式，要求指向一个用户可以看懂的问题类型页面。
* 概要 `title` -- 使用自然语言简单描述问题类型，一般是给工程师看的，偏技术性语言。
* 状态 `status` -- HTTP 状态码，最小 `100` ，最大 `600` 。具体的 Status Code 列表可以参考 <https://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html>
* 具体描述 `detail` -- 关于本次发生的问题的具体的、特定的描述。语言上要求尽可能的非技术化。
* 实例 `instance` -- 同样是一个 URI 形式的字符串，用来指向一个可以解释本次发生的错误的页面

有兴趣的同学可以参考 Problem 的 Schema 定义： <https://zalando.github.io/problem/schema.yaml>

```yml
Problem:
  type: object
  properties:
    type:
      type: string
      format: uri
      description: |
        An absolute URI that identifies the problem type.  When dereferenced,
        it SHOULD provide human-readable documentation for the problem type
        (e.g., using HTML).
      default: 'about:blank'
      example: 'https://zalando.github.io/problem/constraint-violation'
    title:
      type: string
      description: |
        A short, summary of the problem type. Written in english and readable
        for engineers (usually not suited for non technical stakeholders and
        not localized); example: Service Unavailable
    status:
      type: integer
      format: int32
      description: |
        The HTTP status code generated by the origin server for this occurrence
        of the problem.
      minimum: 100
      maximum: 600
      exclusiveMaximum: true
      example: 503
    detail:
      type: string
      description: |
        A human readable explanation specific to this occurrence of the
        problem.
      example: Connection to database timed out
    instance:
      type: string
      format: uri
      description: |
        An absolute URI that identifies the specific occurrence of the problem.
        It may or may not yield further information if dereferenced.
```

### 在工程中集成 Problem 类库

我们使用了一个 zalando 团队提供的在 Spring 使用 Problem 的类库  <https://github.com/zalando/problem-spring-web> 。要在我们的工程中使用的话，需要在 `api/build.gradle` 中增加如下依赖，由于要使用 Jackson Afterburner 模块作为 `json` 的序列化和反序列化的类库配合 Problem ，所以也添加了 `jackson-module-afterburner` 。在统一异常处理时，我们使用 `@ControllerAdvice` ，需要我们添加 `spring-boot-starter-aop`

```groovy
implementation("org.zalando:problem-spring-web:0.20.1")
implementation("com.fasterxml.jackson.module:jackson-module-afterburner")
implementation("org.springframework.boot:spring-boot-starter-aop")
```

为 Problem 配置 Jackson 模块需要在 `config` 下新建 `JacksonConfig.java`

```java
package dev.local.gtm.api.config;

import com.fasterxml.jackson.module.afterburner.AfterburnerModule;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.zalando.problem.ProblemModule;
import org.zalando.problem.validation.ConstraintViolationProblemModule;

/**
 * 为 zalando problem 配置 Jackson
 *
 * @author Peng Wang (wpcfan@gmail.com)
 */
@Configuration
public class JacksonConfig {
    /*
     * 使用 Jackson Afterburner 模块加速 序列化和反序列化过程
     */
    @Bean
    public AfterburnerModule afterburnerModule() {
        return new AfterburnerModule();
    }

    /*
     * 用于序列化和反序列化 RFC7807 Problem 对象的模块。
     */
    @Bean
    ProblemModule problemModule() {
        return new ProblemModule();
    }

    /*
     * 用于序列化和反序列化 ConstraintViolationProblem 的模块
     */
    @Bean
    ConstraintViolationProblemModule constraintViolationProblemModule() {
        return new ConstraintViolationProblemModule();
    }
}
```

### 构建统一异常处理

我们需要在 `dev/local/gtm/api/web` 下新建一个包 `exception` ，在此包下新建一个 `ExceptionTranslator.java`

```java
package dev.local.gtm.api.web.exception;

import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.validation.BindingResult;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.context.request.NativeWebRequest;
import org.zalando.problem.DefaultProblem;
import org.zalando.problem.Problem;
import org.zalando.problem.ProblemBuilder;
import org.zalando.problem.spring.web.advice.ProblemHandling;
import org.zalando.problem.spring.web.advice.validation.ConstraintViolationProblem;

import javax.servlet.http.HttpServletRequest;
import java.util.List;
import java.util.stream.Collectors;

/**
 * 对错误异常进行统一处理，以统一格式输出
 *
 * @author Peng Wang (wpcfan@gmail.com)
 */
@ControllerAdvice
@RequiredArgsConstructor
public class ExceptionTranslator implements ProblemHandling {

    private final HttpServletRequest request;

    @Override
    public ResponseEntity<Problem> process(ResponseEntity<Problem> entity) {
        if (entity.getBody() == null) {
            return entity;
        }
        Problem problem = entity.getBody();
        if (!(problem instanceof ConstraintViolationProblem || problem instanceof DefaultProblem)) {
            return entity;
        }
        ProblemBuilder builder = Problem.builder()
                .withType(Problem.DEFAULT_TYPE.equals(problem.getType()) ? ErrorConstants.DEFAULT_TYPE : problem.getType())
                .withStatus(problem.getStatus())
                .withTitle(problem.getTitle())
                .with("path", request.getRequestURI());

        if (problem instanceof ConstraintViolationProblem) {
            builder
                    .with("violations", ((ConstraintViolationProblem) problem).getViolations())
                    .with("message", ErrorConstants.ERR_VALIDATION);
            return new ResponseEntity<>(builder.build(), entity.getHeaders(), entity.getStatusCode());
        } else {
            builder
                    .withCause(((DefaultProblem) problem).getCause())
                    .withDetail(problem.getDetail())
                    .withInstance(problem.getInstance());
            problem.getParameters().forEach(builder::with);
            if (!problem.getParameters().containsKey("message") && problem.getStatus() != null) {
                builder.with("message", "error.http." + problem.getStatus().getStatusCode());
            }
            return new ResponseEntity<>(builder.build(), entity.getHeaders(), entity.getStatusCode());
        }
    }

    @Override
    public ResponseEntity<Problem> handleMethodArgumentNotValid(MethodArgumentNotValidException ex, NativeWebRequest req) {
        BindingResult result = ex.getBindingResult();
        List<FieldErrorVM> fieldErrors = result.getFieldErrors().stream()
                .map(f -> new FieldErrorVM(f.getObjectName(), f.getField(), f.getCode()))
                .collect(Collectors.toList());

        Problem problem = Problem.builder()
                .withType(ErrorConstants.CONSTRAINT_VIOLATION_TYPE)
                .withTitle("方法参数不正确")
                .withStatus(defaultConstraintViolationStatus())
                .with("message", ErrorConstants.ERR_VALIDATION)
                .with("fieldErrors", fieldErrors)
                .build();
        return create(ex, problem, req);
    }
}
```

首先我们实现了 `ProblemHandling` 接口并覆写了 `process` 方法和 `handleMethodArgumentNotValid` 方法。其中 process 方法是通用的问题处理方法，我们在其中根据是验证器错误还是 HTTP 错误而构造不同的 `Problem` 。

在 `handleMethodArgumentNotValid` 方法中，我们构造了一个更具体针对方法参数错误的 Problem 对象。方法的参数错误属于 Spring 绑定产生的错误 -- `FieldError` ，这个 `FieldError` 中的很多属性我们并不关心，所以我们需要定义一个 `FieldErrorVM` 用来取出特定的属性，并构建到 Problem 中去 -- `FieldError` 中的 `objectName` 作为 `FieldErrorVM` 中的 `field` ；`FieldError` 中的 `field` 作为 `FieldErrorVM` 中的 `code` 作为 `FieldErrorVM` 中的 `message` 。

```java
@Value
public class FieldErrorVM implements Serializable {

    private static final long serialVersionUID = 1L;

    private final String objectName;

    private final String field;

    private final String message;
}
```

而 `ExceptionTranslator` 类上的 `@ControllerAdvice` 注解让这个类成为了所有 Controller 错误的全局处理类。

### 构建自定义异常

我们构建了很多自定义异常，但这些异常在我们实现了统一异常管理之后都变得非常简单，所以下面我们只分析其中一个。

```java
package dev.local.gtm.api.web.exception;

import org.zalando.problem.AbstractThrowableProblem;
import org.zalando.problem.Status;

/**
 * 密码错误异常
 *
 * @author Peng Wang (wpcfan@gmail.com)
 */
public class InvalidPasswordException extends AbstractThrowableProblem {

    public InvalidPasswordException() {
        super(ErrorConstants.INVALID_PASSWORD_TYPE, "密码不正确", Status.BAD_REQUEST);
    }
}
```

所有的自定义异常均继承 `AbstractThrowableProblem` 这个抽象类。然后基本需要做的就是实现一个构造函数，在其中调用父类的构造。 `AbstractThrowableProblem` 有很多构造函数，总体目标是要构造出一个 Problem，所以我们就给出问题的类型、概要描述和状态等（具体参考前面提到的问题定义）。

```java
public abstract class AbstractThrowableProblem extends ThrowableProblem {

    private final URI type;
    private final String title;
    private final StatusType status;
    private final String detail;
    private final URI instance;
    private final Map<String, Object> parameters;

    protected AbstractThrowableProblem() {
        this(null);
    }

    protected AbstractThrowableProblem(@Nullable final URI type) {
        this(type, null);
    }

    protected AbstractThrowableProblem(@Nullable final URI type,
            @Nullable final String title) {
        this(type, title, null);
    }

    protected AbstractThrowableProblem(@Nullable final URI type,
            @Nullable final String title,
            @Nullable final StatusType status) {
        this(type, title, status, null);
    }

    protected AbstractThrowableProblem(@Nullable final URI type,
            @Nullable final String title,
            @Nullable final StatusType status,
            @Nullable final String detail) {
        this(type, title, status, detail, null);
    }

    protected AbstractThrowableProblem(@Nullable final URI type,
            @Nullable final String title,
            @Nullable final StatusType status,
            @Nullable final String detail,
            @Nullable final URI instance) {
        this(type, title, status, detail, instance, null);
    }

    protected AbstractThrowableProblem(@Nullable final URI type,
            @Nullable final String title,
            @Nullable final StatusType status,
            @Nullable final String detail,
            @Nullable final URI instance,
            @Nullable final ThrowableProblem cause) {
        this(type, title, status, detail, instance, cause, null);
    }

    protected AbstractThrowableProblem(@Nullable final URI type,
            @Nullable final String title,
            @Nullable final StatusType status,
            @Nullable final String detail,
            @Nullable final URI instance,
            @Nullable final ThrowableProblem cause,
            @Nullable final Map<String, Object> parameters) {
        super(cause);
        this.type = Optional.ofNullable(type).orElse(DEFAULT_TYPE);
        this.title = title;
        this.status = status;
        this.detail = detail;
        this.instance = instance;
        this.parameters = Optional.ofNullable(parameters).orElseGet(LinkedHashMap::new);
    }
    // 省略
}
```

我们的 `ErrorConstants.java` 中定义了问题类型的 URI，但这里我们并不去让 URI 真正可访问，在真正的工作中，我们会有专门的文档团队来配合我们，这里就不花时间去写这些文档了。

```java
package dev.local.gtm.api.web.exception;

import java.net.URI;

/**
 * 错误常量定义，使用 uri 形式唯一标识错误类型
 *
 * @author Peng Wang (wpcfan@gmail.com)
 */
public final class ErrorConstants {

    public static final String ERR_CONCURRENCY_FAILURE = "error.concurrencyFailure";
    static final String ERR_VALIDATION = "error.validation";
    private static final String PROBLEM_BASE_URL = "http://www.twigcodes.com/problem";
    static final URI DEFAULT_TYPE = URI.create(PROBLEM_BASE_URL + "/problem-with-message");
    static final URI CONSTRAINT_VIOLATION_TYPE = URI.create(PROBLEM_BASE_URL + "/constraint-violation");
    public static final URI PARAMETERIZED_TYPE = URI.create(PROBLEM_BASE_URL + "/parameterized");
    static final URI INVALID_PASSWORD_TYPE = URI.create(PROBLEM_BASE_URL + "/invalid-password");
    static final URI EMAIL_EXISTED_TYPE = URI.create(PROBLEM_BASE_URL + "/email-already-used");
    static final URI LOGIN_EXISTED_TYPE = URI.create(PROBLEM_BASE_URL + "/login-existed");
    static final URI LOGIN_NOT_FOUND_TYPE = URI.create(PROBLEM_BASE_URL + "/login-not-found");
    static final URI MOBILE_EXISTED_TYPE = URI.create(PROBLEM_BASE_URL + "/mobile-already-used");
    static final URI MOBILE_NOT_FOUND_TYPE = URI.create(PROBLEM_BASE_URL + "/mobile-not-found");
    static final URI MOBILE_VERIFICATION_FAILED_TYPE = URI.create(PROBLEM_BASE_URL + "/mobile-verification-failed");
    static final URI RESET_KEY_NOT_MATCH_TYPE = URI.create(PROBLEM_BASE_URL + "/reset-key-not-match");
    static final URI EMAIL_NOT_FOUND_TYPE = URI.create(PROBLEM_BASE_URL + "/email-not-found");

    private ErrorConstants() {
    }
}
```
