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
                        .antMatchers("/v2/api-docs/**").permitAll()
                        .antMatchers("/swagger-resources/configuration/ui").permitAll()
                        .antMatchers("/swagger-ui/index.html").hasAuthority(AuthoritiesConstants.ADMIN)
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

* `@EnableSwagger2` --
* `@ComponentScan(basePackages = "...")`
* `@Import`
* `Docket`
* `ApiInfo`

### 使用注解完善文档

TODO：细化并给出例子

### 文档的效果和使用

TODO：细化并给出例子
