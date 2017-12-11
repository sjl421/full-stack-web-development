# 工程项目的结构

## 前端项目

Todo

## 后端项目

### 多项目构建

在大型软件开发中，一个项目解决所有问题显然不可取的，因为存在太多的开发团队共同协作，所以对项目进行拆分，形成多个子项目的形式是普遍存在的。而且一个子项目只专注自己的逻辑也易于维护和拓展，现在随着容器和微服务的理念逐渐获得大家的认可，多个子项目分别发布到容器和形成多个微服务也逐渐成为趋势。

我们的项目使用 `Gradle` 来处理多项目的构建，包括大项目和子项目的依赖管理以及容器的建立等。对于 `Gradle` 项目来说，我们会有一个根项目，这个根项目下会建立若干子项目，具体文件结构如下：

```
|--spring-boot-tut （根项目）
|----common （共享子项目）
|------src （子项目源码目录）
|--------main （子项目开发源码目录）
|----------java（子项目开发 Java 类源码目录）
|----------resources（子项目资源类源码目录）
|------build.gradle （子项目 gradle 构建文件）
|----api （API 子项目）
|------src
|--------main
|----------java
|----------resources
|------build.gradle
|----report （报表子项目）
|------src
|--------main
|----------java
|----------resources
|------build.gradle
|--build.gradle （根项目构建文件）
|--settings.gradle （根项目设置文件）
```

要让 Gradle 支持多项目的话，首先需要把 `settings.gradle` 改成

```
include 'common'
include 'api'
include 'report'
rootProject.name = 'spring-boot-tut'
```

这样 `spring-boot-tut` 就成为了根项目，而 `common` 、`api` 和 `report` 就是其之下的子项目。接下来，我们看一下根项目的 `build.gradle`，对于多项目构建来说，根项目的 `build.gradle` 中应该尽可能的配置各子项目中共同的配置，从而让子项目只配置自己不同的东西。

```
// 一个典型的根项目的构建文件结构
buildscript {
    /*
     * 构建脚本段落可以配置整个项目需要的插件，构建过程中的依赖以及依赖类库的版本号等
     */
}

allprojects {
    /*
     * 在这个段落中你可以声明对于所有项目（含根项目）都适用的配置，比如依赖性的仓储等
     */
}

subprojects {
    /*
     * 在这个段落中你可以声明适用于各子项目的配置（不包括根项目哦）
     */
    version = "0.0.1"
}

/*
 * 对于子项目的特殊配置
 */
project(':common') {

}

project(':api') {

}

project(':report') {

}
```

其中，`buildscript` 段落用于配置 `gradle` 脚本生成时需要的东西，比如配置整个项目需要的插件，构建过程中的依赖以及在其他部分需要引用的依赖类库的版本号等，就像下面这样，我们在 `ext` 中定义了一些变量来集中配置了所有依赖的版本号，无论是根项目还是子项目都可以使用这些变量来指定版本号。这样做的好处是当依赖的版本更新时，我们无需四处更改散落在各处的版本号。此外在这个段落中我们还提供了项目所需的第三方 `Gradle` 插件所需的依赖：`spring-boot-gradle-plugin`、`gradle-docker` 和 `dependency-management-plugin`，这样在后面，各子项目可以简单的使用诸如 `apply plugin: 'io.spring.dependency-management'` 、 `apply plugin: 'docker'` 等即可。

```
buildscript {
    ext {
        springBootVersion = '1.5.4.RELEASE'
        springCtxSupportVersion = '4.2.0.RELEASE'
        lombokVersion = '1.16.16'
        jjwtVersion = '0.7.0'
        jasperVersion = '6.4.0'
        poiVersion = '3.16'
        itextVersion = '2.1.7'
        olap4jVersion = '1.2.0'
        gradleDockerVersion = '1.2'
        gradleDMVersion = '1.0.3.RELEASE'
    }
    repositories {
        jcenter()
    }
    dependencies {
        classpath("org.springframework.boot:spring-boot-gradle-plugin:${springBootVersion}")
        classpath("se.transmode.gradle:gradle-docker:${gradleDockerVersion}")
        classpath("io.spring.gradle:dependency-management-plugin:${gradleDMVersion}")
    }
}
```

`allprojects` 中可以声明对于所有项目（含根项目）都适用的配置，比如依赖性的仓储等。而 `subprojects` 和 `allprojects` 的区别在于 `subprojecrts` 只应用到子项目，而非根项目。所以大部分通用型配置可以通过 `subprojects` 和 `allprojects` 来完成。下面列出的样例配置中，我们为所有的项目包括根项目配置了依赖仓储以及软件的 `group`，同时为每个**子项目**配置了 `java` 和 `idea` 两个插件、版本号和通用的测试依赖。

```
allprojects {
    group = 'spring-tut'
    repositories() {
        jcenter()
    }
}

subprojects {
    apply plugin: 'java'
    apply plugin: 'idea'
    version = "0.0.1"
    dependencies {
        testCompile("org.springframework.boot:spring-boot-starter-test")
    }
}
```

除此之外呢，为了展示一下 `project` 的用法， 我们这个例子里把每个子项目的依赖放到根 `build.gradle` 中的 `project(':子项目名')` 中列出，这样做有好处也有缺点，好处是依赖性的管理统一在根 `build.gradle` 完成，对于依赖的情况一目了然。当然缺点是每个项目更改依赖时都会造成根 `gradle` 的更新，这样的话如果一个项目有非常多的子项目时，会在协作上出现一些问题。所以请根据具体情况决定把依赖放到根 `build.gradle` 中的 `project(':子项目名')` 中还是放到各子项目的 `build.gradle` 中。

```
project(':common') {
    dependencies {
        compile("org.springframework.boot:spring-boot-starter-data-rest")
        compile("org.springframework.boot:spring-boot-starter-data-mongodb")
        compile("org.projectlombok:lombok:${lombokVersion}")
    }
}

project(':api') {
    dependencies {
        compile project(':common')
        compile("org.springframework.boot:spring-boot-devtools")
        compile("org.springframework.boot:spring-boot-starter-security")
        compile("io.jsonwebtoken:jjwt:${jjwtVersion}")
        compile("org.projectlombok:lombok:${lombokVersion}")
    }
}

project(':report') {
    dependencies {
        compile project(':common')
        compile("org.springframework.boot:spring-boot-devtools")
        // the following 5 are required by jasperreport rendering
        compile files(["lib/simsun.jar"])
        compile("org.springframework.boot:spring-boot-starter-web")
        compile("org.springframework:spring-context-support:${springCtxSupportVersion}")
        compile("net.sf.jasperreports:jasperreports:${jasperVersion}")
        compile("com.lowagie:itext:${itextVersion}")
        compile("org.apache.poi:poi:${poiVersion}")
        compile("org.olap4j:olap4j:${olap4jVersion}")
    }
}
```



