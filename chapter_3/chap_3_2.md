# API 的构建可以如此简单

## API 工程结构

我们 API 是整个后端工程组中的一个子项目，也是我们这个工程的第一个子项目，请按照下面的目录结构方式建立 api 项目目录 -- 在 `gtm-backend` 目录下建立一个子目录 `api`，然后建立对应的目录和文件。

```bash
|--gtm-backend （根项目）
|----api （API 子项目）
|------src
|--------main
|----------java
|----------resources
|------build.gradle
```

然后将 `api` 目录下的 `build.gradle` 修改成下面的样子

```groovy
apply plugin: 'org.springframework.boot'

dependencies {
    implementation('org.springframework.boot:spring-boot-starter-web')
}
```

同时，请确保父一级的 `build.gradle` 已经更新成下面的样子

```groovy
buildscript {
    ext {
        springBootVersion = '2.0.0.RELEASE'
        gradleDockerVersion = '1.2'
    }
    repositories {
        maven { setUrl('http://maven.aliyun.com/nexus/content/groups/public/') }
        maven { setUrl('http://maven.aliyun.com/nexus/content/repositories/jcenter') }
    }
    dependencies {
        classpath("org.springframework.boot:spring-boot-gradle-plugin:${springBootVersion}")
        classpath("se.transmode.gradle:gradle-docker:${gradleDockerVersion}")
    }
}

allprojects {
    group = 'dev.local.gtm'
    repositories {
        maven { setUrl('http://maven.aliyun.com/nexus/content/groups/public/') }
        maven { setUrl('http://maven.aliyun.com/nexus/content/repositories/jcenter') }
    }
}

subprojects {
    version = "0.0.1"
    tasks.withType(Jar) {
        baseName = "$project.name-$version"
    }
    apply plugin: 'java'
    apply plugin: 'idea'
    apply plugin: 'eclipse'
    apply plugin: 'io.spring.dependency-management'
    sourceCompatibility = 1.8
    targetCompatibility = 1.8
    dependencies {
        testImplementation("org.springframework.boot:spring-boot-starter-test")
    }
}
```

并且更新 `gtm-backend` 目录下的 `settings.gradle` 以便包含 `api` 作为子项目

```groovy
include 'api'
rootProject.name = 'gtm-backend'
```

## 领域对象

那么我们的源代码目录在哪里呢？我们得手动建立一个，这个目录一般情况下是 `src/main/java`。好的，下面我们要开始第一个 RESTful 的API搭建了，首先还是在 `src/main/java` 下新建一个 `package`。既然是本机的就叫 `dev.local.gtm` 吧。我们还是来尝试建立一个 Web API，在 `dev.local.gtm.api` 下建立一个子 `package`: `domain`，然后创建一个 Task 的领域对象：

```java
package dev.local.gtm.api.domain;

/**
 * Task 是一个领域对象（domain object）
 */
public class Task {
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

这个对象很简单，只是描述了 Task 的几个属性： `id` 、 `desc` 和 `completed` 。我们的 API 返回或接受的参数就是以这个对象为模型的类或集合。

## 构造 Controller

我们经常看到的 RESTful API 是这样的：`http://local.dev/tasks`、`http://local.dev/tasks/1` 。而 Controller 就是要暴露这样的 API 给外部使用。现在我们同样的在 `task` 下建立一个叫 `TaskController` 的 java 文件:

```java
package dev.local.gtm.api.controller;

import dev.local.gtm.api.domain.Task;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.ArrayList;
import java.util.List;

/**
 * 使用 @RestController 来标记这个类是个 Controller
 */
@RestController
public class TaskController {
    // 使用@RequstMapping指定可以访问的URL路径
    @RequestMapping("/tasks")
    public List<Task> getAllTasks() {
        List<Task> tasks = new ArrayList<>();
        Task item1 = new Task();
        item1.setId("1");
        item1.setCompleted(false);
        item1.setDesc("go swimming");
        tasks.add(item1);
        Task item2 = new Task();
        item2.setId("2");
        item2.setCompleted(true);
        item2.setDesc("go for lunch");
        tasks.add(item2);
        return tasks;
    }
}

```

上面这个文件也比较简单，但注意到以下几个事情：

* `@RestController` 和 `@RequestMapping` 这两个是注解，这种方式原来在 .Net 中很常见，后来 Java 也引进过来。一方面它们可以增加代码的可读性，另一方面也有效减少了代码的编写。具体机理就不讲了，简单来说就是利用 Java 的反射机制和 IoC 模式结合把注释的特性或属性注入到被注释的对象中。
* 我们看到 `List<Task> getAllTasks()` 方法中简单的返回了一个 List ，并未做任何转换成 json 对象的处理，这个是 Spring 会自动利用 `Jackson` 这个类库的方法将其转换成了 json 。

我们到这就基本接近成功了，但是现在缺少一个入口，那么在 `dev.local.gtm.api` 包下面建立一个 `Applicaiton.java` 吧。

```java
package dev.local.gtm.api;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

/**
 * Spring Boot 应用的入口文件
 */
@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}

```

同样的我们只需简单的标注这个类是 `@SpringBootApplication` 就一切 ok 了，习惯了 Spring 繁琐的 xml 配置的同学可能需要适应一下。

## 启动服务

然后可以使用下面任意一种方法去启动我们的 Web 服务：

 1. 命令行中： `./gradlew bootRun`
 2. IDEA 中在 `Application` 中右键选择 `Run 'Application'`

![在 IDE 中选择 Run Application](/assets/2018-04-01-17-26-40.png)

如果使用命令行的话，可以输入下面的命令启动 API 服务。

```bash
./gradlew :api:bootRun
```

注意上面的命令是在 `gtm-backend` 根目录执行，而我们的 API 是整个项目的一个子项目，所以采用 `:api` 这种方式指明是哪个子项目，然后使用 `:bootRun` 指定执行什么任务。所以总结起来，运行命令就是下面这个样子

```bash
./gradlew :<子项目名称>:<任务名称>
```

执行命令后，如果你可以看到类似下面的输出，那就说明服务启动成功了

```log
Task :api:bootRun

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.0.0.RELEASE)

2018-04-02 09:53:09.506  INFO 48808 --- [           main] dev.local.gtm.api.Application            : Starting Application on wangpengdeMacBook-Pro.local with PID 48808 (/Users/wangpeng/workspace/books/gtm-backend/api/build/classes/java/main started by wangpeng in /Users/wangpeng/workspace/books/gtm-backend/api)
2018-04-02 09:53:09.511  INFO 48808 --- [           main] dev.local.gtm.api.Application            : No active profile set, falling back to default profiles: default
2018-04-02 09:53:09.574  INFO 48808 --- [           main] ConfigServletWebServerApplicationContext : Refreshing org.springframework.boot.web.servlet.context.AnnotationConfigServletWebServerApplicationContext@c8e4bb0: startup date [Mon Apr 02 09:53:09 CST 2018]; root of context hierarchy
2018-04-02 09:53:10.926  INFO 48808 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8080 (http)
2018-04-02 09:53:10.958  INFO 48808 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2018-04-02 09:53:10.959  INFO 48808 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet Engine: Apache Tomcat/8.5.28
2018-04-02 09:53:10.968  INFO 48808 --- [ost-startStop-1] o.a.catalina.core.AprLifecycleListener   : The APR based Apache Tomcat Native library which allows optimal performance in production environments was not found on the java.library.path: [/Users/wangpeng/Library/Java/Extensions:/Library/Java/Extensions:/Network/Library/Java/Extensions:/System/Library/Java/Extensions:/usr/lib/java:.]
2018-04-02 09:53:11.067  INFO 48808 --- [ost-startStop-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2018-04-02 09:53:11.068  INFO 48808 --- [ost-startStop-1] o.s.web.context.ContextLoader            : Root WebApplicationContext: initialization completed in 1497 ms
2018-04-02 09:53:11.192  INFO 48808 --- [ost-startStop-1] o.s.b.w.servlet.ServletRegistrationBean  : Servlet dispatcherServlet mapped to [/]
2018-04-02 09:53:11.197  INFO 48808 --- [ost-startStop-1] o.s.b.w.servlet.FilterRegistrationBean   : Mapping filter: 'characterEncodingFilter' to: [/*]
2018-04-02 09:53:11.198  INFO 48808 --- [ost-startStop-1] o.s.b.w.servlet.FilterRegistrationBean   : Mapping filter: 'hiddenHttpMethodFilter' to: [/*]
2018-04-02 09:53:11.198  INFO 48808 --- [ost-startStop-1] o.s.b.w.servlet.FilterRegistrationBean   : Mapping filter: 'httpPutFormContentFilter' to: [/*]
2018-04-02 09:53:11.198  INFO 48808 --- [ost-startStop-1] o.s.b.w.servlet.FilterRegistrationBean   : Mapping filter: 'requestContextFilter' to: [/*]
2018-04-02 09:53:11.545  INFO 48808 --- [           main] s.w.s.m.m.a.RequestMappingHandlerAdapter : Looking for @ControllerAdvice: org.springframework.boot.web.servlet.context.AnnotationConfigServletWebServerApplicationContext@c8e4bb0: startup date [Mon Apr 02 09:53:09 CST 2018]; root of context hierarchy
2018-04-02 09:53:11.640  INFO 48808 --- [           main]
<=========----> 75% EXECUTING [11m 11s]
> :api:bootRun
> IDLE
> IDLE
> IDLE

```

启动后，打开浏览器访问 `http://localhost:8080/tasks` 就可以看到我们的 `json` 形式的返回结果了。

![访问 API 地址](/assets/2018-04-02-09-54-46.png)

是不是有种感觉，做一个 API 也太 easy 了？对了，Spring Boot 的一个主要作用就是将我们从繁琐的配置和大量模式化代码中解放出来。

## 测试 API

上面的测试只是通过浏览器测试一个简单的 `GET` 请求，但复杂一些的请求就没办法这么测试了。“工欲善其事，必先利其器”，我们得有一个趁手的测试工具才好。`*nix` 用户可能已经比较熟悉 `curl` ，对于新手或者不熟悉 `curl` 的同学，我这里推荐 Postman，一个有图形化界面的专业 HTTP API 测试工具，支持 Mac/Linux/Windows，大家可以去 <https://www.getpostman.com/apps> 下载对应平台的版本。

Postman 的界面非常友好，基本上看一下就知道大概怎么用了，我在下面的截图中也标识了大部分测试 API 中最常见的一些参数配置区域和返回的显示区域。

![Postman 界面](/assets/2018-04-02-13-34-09.png)

从图中也可以很清晰的看到我们的第一个 API 返回的结果了，由于 Postman 会很智能识别返回的格式（这个例子里面是 `json` ），这个显示的效果比浏览器可是好太多了。
