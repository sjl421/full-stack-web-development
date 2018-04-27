# 跨域以及 API 文档

## 跨域解决方案 -- CORS

前面我们初步做出了一个可以实现受保护的 REST API ，但是我们没有涉及一个前端领域很重要的问题，那就是 **跨域请求**（ `cross-origin HTTP request` ）。先来回顾一些背景知识：

### 什么是跨域？

> 定义：当我们从本身站点请求不同域名或端口的服务所提供的资源时，就会发起跨域请求。

例如最常见的我们很多的 `css` 样式文件是会链接到某个公共 CDN 服务器上，而不是在本身的服务器上，这其实就是典型的一个跨域请求。但浏览器由于安全原因限制了在脚本（ `script` ）中发起的跨域 HTTP 请求。也就是说 `XMLHttpRequest` 和 `Fetch` 等是遵循“同源规则”的，即只能访问自己服务器的指定端口的资源（同一服务器不同端口也会视为跨域）。但这种限制在今天，我们的应用需要访问多种外部 API 或资源的时候就不能满足开发者的需求了，因此就产生了若干对于跨域的解决方案， `JSONP` 是其中一种，但在今天来看主流的更彻底的解决方案是 `CORS`  -- `Cross-Origin Resource Sharing` 。

### 跨域资源共享（ CORS ）

这种机制将跨域的访问控制权交给服务器，这样可以保证安全的跨域数据传输。现代浏览器一般会将 CORS 的支持封装在 HTTP API 之中（ 比如 `XMLHttpRequest` 和 `Fetch` ），这样可以有效控制使用跨域请求的风险，因为你绕不过去，总得要使用 API 吧。

概括来说，这个机制是增加一系列的 HTTP 头来让服务器可以描述哪些源是允许使用浏览器来访问资源的。而且对于简单的请求和复杂请求，处理机制是不一样的。

简单请求仅允许三个 HTTP 方法：GET，POST 以及 HEAD ，另外只能支持若干 header 参数： `Accept` ， `Accept-Language` ， `Content-Language` ， `Content-Type` （值只能是 `application/x-www-form-urlencoded`、`multipart/form-data` 和 `text/plain`）， `DPR` ， `Downlink` ， `Save-Data` ， `Viewport-Width` 和  `Width` 。

对于简单请求来说，比如下面这样一个简单的 GET 请求：从 `http://me.domain` 发起到 <http://another.domain/data/blablabla> 的资源请求

```
GET /data/blablabla/ HTTP/1.1
// 请求的域名
Host: another.domain
...//省略其它部分，重点是下面这句，说明了发起请求者的来源
Origin: http://me.domain
```

应用了 `CORS` 的对方服务器返回的响应应该像下面这个样子，当然这里 `Access-Control-Allow-Origin: *` 中的 `*` 表示任何网站都可以访问该资源，如果要限制只能从 `me.domain` 访问，那么需要改成  `Access-Control-Allow-Origin: http://me.domain`

```txt
HTTP/1.1 200 OK
...//省略其它部分
Access-Control-Allow-Origin: *
...//省略其它部分
Content-Type: application/json
```

那么对于复杂请求怎么办呢？这需要一次预检请求和一次实际的请求，也就是说需要两次和对方服务器的请求/响应。预检请求是以 OPTION 方法进行的，因为 OPTION 方法不会改变任何资源，所以这个预检请求是安全的，它的职责在于发送实际请求将会使用的 HTTP 方法以及将要发送的 HEADER 中将携带哪些内容，这样对方服务器可以根据预检请求的信息决定是否接受。

```txt
// 预检请求
OPTIONS /resources/post/ HTTP/1.1
Host: another.domain
...// 省略其它部分
Origin: http://me.domain
Access-Control-Request-Method: POST
Access-Control-Request-Headers: Content-Type
```

服务器对预检请求的响应如下：

```txt
HTTP/1.1 200 OK
// 省略其它部分
Access-Control-Allow-Origin: http://me.domain
Access-Control-Allow-Methods: POST, GET, OPTIONS
Access-Control-Allow-Headers: Content-Type
Access-Control-Max-Age: 86400
// 省略其它部分
Content-Type: text/plain
```

接下来的正式请求就和上面的简单请求差不多了，就不赘述了。

### Spring Boot 中如何启用 CORS

啰嗦了这么多，终于进入正题，但我一直觉得不能光知其然而不知其所以然，所以各位就忍了吧。加入 CORS 的支持在 Spring Boot 中简单到不忍直视，添加一个配置类即可：

```java
package dev.local.gtm.api.config;

import lombok.RequiredArgsConstructor;
import lombok.extern.log4j.Log4j2;
import lombok.val;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.cors.UrlBasedCorsConfigurationSource;
import org.springframework.web.filter.CorsFilter;

@Log4j2
@RequiredArgsConstructor
@Configuration
public class CorsConfig {

    private final AppProperties appProperties;

    @Bean
    public CorsFilter corsFilter() {
        val source = new UrlBasedCorsConfigurationSource();
        val config = appProperties.getCors();
        if (config.getAllowedOrigins() != null && !config.getAllowedOrigins().isEmpty()) {
            log.debug("注册 CORS 过滤器");
            config.addAllowedOrigin("*");
            config.addAllowedMethod("*");
            config.addAllowedHeader("*");
            source.registerCorsConfiguration("/api/**", config);
            source.registerCorsConfiguration("/management/**", config);
            source.registerCorsConfiguration("/v2/api-docs", config);
            source.registerCorsConfiguration("/*/api/**", config);
            source.registerCorsConfiguration("/*/management/**", config);
        }
        return new CorsFilter(source);
    }
}
```

然后在 `SecurityConfig` 中配置我们建立的 `CorsFilter` 就可以了，这样我们的前端就可以访问 API 了。

```java
@RequiredArgsConstructor
@Configuration
@EnableWebSecurity
@EnableGlobalMethodSecurity(prePostEnabled = true, securedEnabled = true)
@Import(SecurityProblemSupport.class)
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    // 省略其他成员变量
    private final CorsFilter corsFilter;
    // 省略其他方法
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
                .addFilterBefore(corsFilter, UsernamePasswordAuthenticationFilter.class) // <--- 将 corsFilter 配置在 UsernamePasswordAuthenticationFilter 之前
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
                    .authorizeRequests()
                        .antMatchers("/api/auth/**").permitAll()
                        .antMatchers("/api/**").authenticated()
                        .antMatchers("/websocket/tracker").hasAuthority(AuthoritiesConstants.ADMIN)
                        .antMatchers("/websocket/**").permitAll()
                        .antMatchers("/management/health").permitAll()
                        .antMatchers("/management/**").hasAuthority(AuthoritiesConstants.ADMIN)
                .and()
                    .apply(jwtConfigurer);
    }
}
```

## API 文档

大家前面使用 Postman 的时候是不是感觉各种 API 的构造有点麻烦？想象一下，如果有四个团队：后端、前端、iOS 客户端和 Android 客户端。后端开发完 API 之后，其他几个团队怎么知道 API 应该怎么使用呢？当然可以使用传统的文档，但传统文档有几个明显弱点

1. 没有交互性 -- API 文档阅读时很少会引发人提出什么问题，因为 API 是前端或者客户端和后端交互的手段，很多问题也是交互时才发现的。
2. 更新不及时 -- API 的重构和修正在系统联调的时候是很频繁的，文档的更新往往不够及时。
3. 错误率较高 -- 写文档时很多时候会有笔误或者有遗漏，这导致可能后端认为很简单的事情，前端调试很久才发现是文档错了。

因此在这个敏捷开发盛行的时代，我们需要一种敏捷的、可交互的文档来描述 API。 Swagger <https://swagger.io/> 便是这种文档类型中的佼佼者。Swagger 是一种 Rest API 的简单但强大的表现形式，它是标准的和编程语言无关的，不但人可以阅读，而且机器可读，也就是可以直接进行 API 交互。所以它既可以作为 Rest API 的交互式文档，也可以作为 Rest API 的形式化的接口描述，甚至可以直接生成客户端和服务端的代码。

而能将 Swagger 和 Spring Boot 平滑的对接起来的一个开源类库就是 SpringFox <https://github.com/springfox/springfox>

### 配置 Swagger

首先需要在 `api/build.gradle` 中配置 SpringFox 的依赖项，共有三个依赖 `springfox-swagger2` 、 `springfox-bean-validators` 和 `springfox-swagger-ui`

```groovy
// api/build.gradle
// 省略其他部分
dependencies {
    implementation("io.springfox:springfox-swagger2:${springFoxVersion}")
    implementation("io.springfox:springfox-bean-validators:${springFoxVersion}")
    implementation("io.springfox:springfox-swagger-ui:${springFoxVersion}")
    implementation("org.springframework.boot:spring-boot-starter-undertow")
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("io.jsonwebtoken:jjwt:0.9.0")
    implementation("org.springframework.boot:spring-boot-starter-security")
    implementation("org.springframework.boot:spring-boot-starter-aop")
    implementation("org.zalando:problem-spring-web:0.20.1")
    implementation("com.fasterxml.jackson.module:jackson-module-afterburner")
    implementation("org.springframework.boot:spring-boot-starter-data-mongodb")
    testImplementation("org.springframework.security:spring-security-test")
}
```

当然也需要在根项目的 `build.gradle` 中的 `ext` 中指定版本号。

```groovy
buildscript {
    ext {
        // 省略
        springFoxVersion = '2.8.1-SNAPSHOT'
    }
    // 省略
}
```

配置完类库依赖之后呢，我们就可以在 `config` 包下新建一个 `SwaggerConfig.java`

```java
package dev.local.gtm.api.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Import;
import org.springframework.http.ResponseEntity;
import springfox.documentation.builders.PathSelectors;
import springfox.documentation.builders.RequestHandlerSelectors;
import springfox.documentation.service.ApiInfo;
import springfox.documentation.service.Contact;
import springfox.documentation.spi.DocumentationType;
import springfox.documentation.spring.web.plugins.Docket;
import springfox.documentation.swagger2.annotations.EnableSwagger2;

import java.time.LocalDate;
import java.util.Collections;

/**
 * 配置 Swagger 以提供 API 文档
 *
 * @author Peng Wang (wpcfan@gmail.com)
 */
@EnableSwagger2
@ComponentScan(basePackages = "dev.local.gtm.api.web.rest")
@Import({
        springfox.bean.validators.configuration.BeanValidatorPluginsConfiguration.class
})
@Configuration
public class SwaggerConfig {
    /**
     * 配置 Swagger 扫描哪些 API （不列出那些监控 API）
     *
     * @return Docket
     */
    @Bean
    public Docket apiDoc() {
        return new Docket(DocumentationType.SWAGGER_2)
                .select()
                    .apis(RequestHandlerSelectors.basePackage("dev.local.gtm.api.web.rest"))
                    .paths(PathSelectors.any())
                    .build()
                .pathMapping("/")
                .directModelSubstitute(LocalDate.class, String.class)
                .genericModelSubstitutes(ResponseEntity.class)
                .apiInfo(apiInfo());
    }

    /**
     * 对 API 的概要信息进行定制
     *
     * @return ApiInfo
     */
    private ApiInfo apiInfo() {
        return new ApiInfo(
                "GTM API 文档",
                "所有 GTM 开放的 API 接口，供 Android， iOS 和 Web 客户端调用",
                "1.0",
                "http://www.twigcodes.com/gtm/tos.html",
                new Contact("Peng Wang", "http://www.twigcodes.com", "wpcfan@gmail.com"),
                "API 授权协议", "http://www.twigcodes.com/gtm/api-license.html", Collections.emptyList());
    }
}
```

这段配置代码要解释几个地方

* `@EnableSwagger2` -- 这个注解就可以将 Swagger 集成进来
* `@ComponentScan(basePackages = "...")` -- 指明我们需要扫描哪些包下的 Controller
* `@Import` -- 如果要使用 Bean 验证，需要导入 SpringFox 的验证类。
* `Docket` -- 这个是 SpringFox 的基础配置类
* `ApiInfo` -- 对于 API 文档基础信息，比如版本号，团队信息等进行配置。

现在为了可以看到文档效果，我们还需要在 `SecurityConfig` 中加入 Swagger UI 的支持，默认情况下 Swagger 集成了一个自己的 UI ，它的入口 `URL` 一般是 `/swagger-ui.html` ，我们暂时允许匿名访问该 `URL` 。

```java
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    // 省略

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
                .addFilterBefore(corsFilter, UsernamePasswordAuthenticationFilter.class)
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
                    .authorizeRequests()
                        .antMatchers("/api/auth/**").permitAll()
                        .antMatchers("/api/**").authenticated()
                        .antMatchers("/websocket/tracker").hasAuthority(AuthoritiesConstants.ADMIN)
                        .antMatchers("/websocket/**").permitAll()
                        .antMatchers("/management/health").permitAll()
                        .antMatchers("/management/**").hasAuthority(AuthoritiesConstants.ADMIN)
                        .antMatchers("/v2/api-docs/**").permitAll()
                        .antMatchers("/swagger-resources/configuration/ui").permitAll()
                         // 这里暂时允许匿名访问 API 文档
                        .antMatchers("/swagger-ui/index.html").permitAll()
                .and()
                    .apply(jwtConfigurer);
    }
}
```

现在我们看看文档的效果，访问 <http://localhost:8080/swagger-ui/index.html> ，可以看到我们的文档首页。里面现在列出了我们现有的两个资源 ( `Controller` )

![Swagger 文档首页](/assets/2018-04-27-11-16-20.png)

那么如果我们展开其中的一项，比如 `GET /api/auth/captcha` ，可以看到如下图所示的界面，包括这个 API 需要的参数，和可能的响应码。

![展开 Auth 资源的 /api/auth/captcha](/assets/2018-04-27-11-18-18.png)

如果我们点击 `Try it out` 按钮，然后点 `Execute` ，那么就可以发送该 API 请求，并得到返回结果。

![得到的 API 返回](/assets/2018-04-27-11-24-53.png)

哈哈，这样我们可以把 Postman 抛弃了，对于前端或客户端开发的同事来说，直接看这个文档的同时就可以进行验证和调试了。

### 对于受保护的 API 的访问

刚刚我们都是对匿名可以访问的 API 进行验证，但如果是我们刚刚建立的受保护的 API，比如需要 `Authorization` 头的这种怎么办呢？SpringFox 提供了 `SecurityContext` 和 `SecurityScheme` 来帮我们实现这个访问安全 API 的目的。

```java
public class SwaggerConfig {
    /**
     * 配置 Swagger 扫描哪些 API （不列出那些监控 API）
     *
     * @return Docket
     */
    @Bean
    public Docket apiDoc() {
        return new Docket(DocumentationType.SWAGGER_2)
                .select()
                    .apis(RequestHandlerSelectors.basePackage("dev.local.gtm.api.web.rest"))
                    .paths(PathSelectors.any())
                    .build()
                .pathMapping("/")
                .securitySchemes(newArrayList(apiKey()))
                .securityContexts(newArrayList(securityContext()))
                .directModelSubstitute(LocalDate.class, String.class)
                .genericModelSubstitutes(ResponseEntity.class)
                .apiInfo(apiInfo());

    }

    private ApiKey apiKey() {
        // 用于Swagger UI测试时添加Bearer Token
        return new ApiKey("Bearer", HttpHeaders.AUTHORIZATION, In.HEADER.name());
    }

    private SecurityContext securityContext() {
        return SecurityContext.builder()
                .securityReferences(defaultAuth())
                .forPaths(PathSelectors.regex("/api/((?!auth).)*"))
                .build();
    }

    private List<SecurityReference> defaultAuth() {
        AuthorizationScope authorizationScope = new AuthorizationScope("global", "accessEverything");
        AuthorizationScope[] authorizationScopes = new AuthorizationScope[1];
        authorizationScopes[0] = authorizationScope;
        return newArrayList(new SecurityReference("Bearer", authorizationScopes));
    }
    // 省略
}
```

* `SecurityScheme` -- 提供一种保护 API 的安全策略，目前内建支持 `ApiKey` ， `BasicAuth` 和 `OAuth` 。由于我们之前实现的是 JWT tokne ，属于 `ApiKey` ，所以下面我们提供的也是 `ApiKey` ，其他实现形式，请查阅官网文档 <http://springfox.github.io/springfox/docs/current/>
* `ApiKey` -- 在下面的配置中我们使用了 `ApiKey("Bearer", HttpHeaders.AUTHORIZATION, In.HEADER.name())` 构造一个 `ApiKey` 。
  * 而 `ApiKey` 构造函数的第一个参数是我们如何命名这个 `ApiKey` ，后面在 `SecurityReference("Bearer", authorizationScopes)` 引用的名字需要和这里定义的一致。
  * 第二个参数指出 ApiKey 的 `key` 是什么，举例来说，我们的 API 鉴权是 `Authorization: Bearer xxxx` 的形式，这个 `key` 就是 `Authorization` ，所以我们使用 `HttpHeaders.AUTHORIZATION` 作为第二个参数。
  * 第三个参数是这个 `key` 和我们传入的 `value` （ `value` 就是 `Bearer xxxx` 部分） 要在请求的什么地方添加，我们需要的是在 Request 头部，所以是 `In.HEADER.name()`
* `SecurityContext` -- 提供一种全局的方式用来选择实施哪种安全策略（安全策略就是上面的 `SecurityScheme` ），在下面的配置中我们以正则表达式的形式 `.forPaths(PathSelectors.regex("/api/((?!auth).)*"))`  对不是 `/api/auth**` 的这种路径实施 ApiKey 的安全策略。也就是说在 API 文档中进行交互式测试时，对于非 `auth` 路径下的 API 添加 `ApiKey` 中定义的 Header 。

那么现在如果重新访问 <http://localhost/swagger-ui/index.html> 的话，我们会看到出现了一个 `Authorize` 按钮。

![授权按钮](/assets/2018-04-27-13-02-37.png)

点击这个按钮，就会出现下面的界面，让你输入 `Authorization` 的值，也就是 `Bearer xxxx` ，这个值你可以通过调用注册或登录的 API 得到，把得到的值前面加上 `Bearer` 和一个空格，粘贴到 `value` 的输入框中，点击 `Authorize` 按钮，然后点 `Close` 关闭对话框。

![授权对话框](/assets/2018-04-27-13-04-24.png)

现在，如果我们访问某个受保护的 API ，就会得到类似下图所示的效果。

![访问受保护 API](/assets/2018-04-27-13-10-25.png)

我们可以看到在文档生成的 `curl` 命令中，使用 `-H` 添加了一个 `Authorization` 的 Header 键值对。有兴趣的同学在 `*nix` 系统上可以开一个 `terminal` 实验下面的命令，得到的输出和我们界面上看到的应该一样。

```bash
curl -X GET "http://localhost:8080/api/tasks" -H "accept: */*" -H "Authorization: Bearer eyJhbGciOiJIUzUxMiJ9.eyJzdWIiOiJ3cGNmYW4iLCJhdXRoIjoiUk9MRV9VU0VSIiwiZXhwIjoxNTI0ODEyODU2fQ.DmV8MF_XP1I99UPTGMewFAYrDzFhSgKbgOh0M0_t4LrxPwynV6BvJaN0rHIGDFE5WeMofrEbvxWsWoKW2dGyAQ"
```

### 使用注解完善文档

SpringFox 提供了一系列注解来帮助我们完善文档，常用的注解如下表所示。

常用注解 | 应用对象 | 举例说明
---|---|---
`@ApiModelProperty` | 领域对象的属性 | `@ApiModelProperty(value="姓名")`
`@ApiParam` | Controller 的参数 | `@ApiParam(value = "用户名") @RequestParam("username") String username`
`@ApiOperation` | Controller 方法 | `@ApiOperation(value = "用户登录鉴权接口",notes = "客户端在 RequestBody 中以 json 传入用户名、密码，如果成功以 json 形式返回该用户信息")`

我们具体看一下在代码中如何使用，下面的代码体现了 `ApiOperation` 和 `@ApiParam`

```java
// 省略
public class AuthResource {

    // 省略
    // 对这个 API 的描述
    @ApiOperation(value = "用户登录鉴权接口",
            notes = "客户端在 RequestBody 中以 json 传入用户名、密码，如果成功以 json 形式返回该用户信息")
    @PostMapping(value = "/auth/login")
    public ResponseEntity<JWTToken> login(@RequestBody final Auth auth) {
        log.debug("REST 请求 -- 将对用户: {} 执行登录鉴权", auth);
        return generateJWTHeader(auth.getLogin(), auth.getPassword());
    }

    // 省略

    @GetMapping("/auth/search/username")
    public ExistCheck usernameExisted(
            //对参数的描述
            @ApiParam(value = "用户名") @RequestParam("username") String username) {
        log.debug("REST 请求 -- 用户名是否存在 {}", username);
        return new ExistCheck(authService.usernameExisted(username));
    }
    // 省略
}
```

由下图可以看出，`@ApiOperation` 的 `value` 指定的是一个简短描述，而 `notes` 可以较详细的说明这个 API 的用途。

![ApiOperation 的表现形式](/assets/2018-04-27-14-32-10.png)

而下图展现了 `@ApiParam` 的效果，就是对 Controller 方法参数的描述。

![ApiParam 的效果](/assets/2018-04-27-14-34-24.png)

而如果使用 `@ApiModelProperty` 对领域对象进行描述的话，我们可以试一下把 `Auth` 改造成下面的样子。

```java
@Data
public class Auth {
    @ApiModelProperty(value = "登录名")
    private String login;
    @ApiModelProperty(value = "密码")
    private String password;
}
```

我们在文档的 `Parameters` 区块下的 `Description` 中，点击 `Example Value` 右边的 `Model` ，就可以看到效果了，是对于领域对象属性的说明。

![ApiModelProperty 的展现](/assets/2018-04-27-14-38-37.png)

API 的文档注解功能还是很强大的，但在实际的使用中个人不是很推荐每个 API 都用注解做详细说明，因为会导致代码的注解部分太长，可读性下降。一般我们只对关键 API 或容易产生误解的部分进行注解就可以了。
