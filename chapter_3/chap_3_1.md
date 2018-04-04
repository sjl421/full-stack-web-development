# 创建一个 Spring Boot 工程

创建一个 Spring Boot 工程有几种方式

* 通过 gradle 或 maven 进行创建
  * 优势：不依赖任何 IDE 或操作系统， 和  各种持续集成、自动化测试  框架结合行程
  * 劣势：学习成本较高，需要对 Gradle 或 Maven 进行团队的培训
* 通过 IDE 进行创建
  * 优势 ：创建过程非常简单
  * 劣势 ：严重依赖 IDE

这里我们会分别给出这两种方式的创建 Spring Boot 工程的  步骤， 虽然我个人更推荐第一种方式。

##  通过 Gradle 创建

本书中的以后章节中都采用 Gradle 来做工程组织 ，这种方式其实和 maven 方式差不多，只不过个人感觉 Gradle 比 Maven 的 `xml` 形式的配置文件  可读性要好很多。

### 安装 Gradle

首先当然要先安装 Gradle，这一步只有第一次当系统中没有 Gradle 的时候才需要，以后的项目创建并不需要这一个步骤。

* Linux: 在 `terminal` 中敲 `sdk install gradle` 。假如你没有安装 `sdkman` 的话，请参考第一章内容进行安装。
* macOS：在 `terminal` 中敲 `brew install gradle` ，或者你也可以按 Linux 的安装方式进行
* Windows：在命令行下执行 `scoop install gradle` ，如果你没有安装 `scoop` ，还是请参考第一章相关内容

要确认是否安装成功，可以在开一个命令行窗口，敲 `gradle -v`，如果你可以看到类似的输出，那么就恭喜你了，安装成功！

```bash
------------------------------------------------------------
Gradle 4.4.0
------------------------------------------------------------

Build time:   2017-10-02 15:36:21 UTC
Revision:     a88ebd6be7840c2e59ae4782eb0f27fbe3405ddf

Groovy:       2.4.12
Ant:          Apache Ant(TM) version 1.9.6 compiled on June 29 2015
JVM:          1.8.0_111 (Oracle Corporation 25.111-b14)
OS:           Mac OS X 10.13.3 x86_64
```

### 创建 Gradle 工程

创建一个 Gradle 只需要两步，再简单不过了。

 1. 新建一个工程目录 `gtm`
 2. 在此目录下使用 Gradle 进行初始化 `gradle init`（就和在 `git` 中使用 `git init` 的效果类似，这一动作初始化的生成一个工程）

这个命令帮我们建立一个一个使用 gradle 进行管理的模版工程，下面的列表中给出了这个命令为我们创建了哪些文件和文件夹，以及它们的作用：

* `build.gradle`：有过 Android 开发经验的童鞋可能觉得很亲切的，这个就是我们用于管理和配置工程的核心文件了。
* `gradlew`：用于 `*nix` 环境下的 gradle wrapper 文件。
* `gradlew.bat`：用于 `Windows` 环境下的 gradle wrapper 文件
* `setting.gradle`：用于管理多项目的 gradle 工程时使用，单项目时可以不做理会。
* `gradle` 目录：wrapper 的 jar 和属性设置文件所在的文件夹。

### 什么是 Wrapper

简单说两句什么是 `gradle wrapper`。你是否有过这样的经历？在安装/编译一个工程时需要一些先决条件，需要安装一些软件或设置一些参数。如果这一切比较顺利还好，但很多时候我们会发现这样那样的问题，比如版本不对，参数没设置等等。`gradle wrapper` 就是这样一个让你不会浪费时间在配置问题上的方案。它会对应一个开发中使用的 gradle 版本，以确保任何人任何时候得到的结果是一致的。

* `./gradlew <task>`:  在 `*nix` 平台上运行，例如 Linux 或 Mac OS X
* `gradlew <task>` 在 Windows 平台运行（是通过 `gradlew.bat` 来执行的）

#### Wrapper 的配置

当我们在 `terminal` 执行 gradle wrapper 生成相关文件的时候，可以为其指定一些参数，来控制wrapper的生成，比如依赖的版本等。很多时候，我们可能需要不同版本的 gradle，比如 Spring Boot 2.0 就需要 Gradle 4.0 以上版本，比如我们要指定使用 `4.4` 版本的话，就可以使用下面的命令。

```bash
gradle wrapper --gradle-version 4.4
```

这个命令其实是更改了 `gradle/wrapper/gradle-wrapper.properties` 这个文件中的 `distributionUrl` ，从下面的文件可以看出我们使用的是 `gradle-4.4-bin.zip`。

```ini
distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists
distributionUrl=https\://services.gradle.org/distributions/gradle-4.4-bin.zip
```

那么接下来还有一个问题就是，有的时候我们可能希望使用的是含 gradle 源代码的版本，也就是文件名带有 `all` 的那种发行版本。这种情况我们可以使用另一个参数 `--distribution-type`

```bash
gradle wrapper --gradle-version 4.4 --distribution-type all
```

这样的命令就会使得 `distributionUrl` 指向一个带有源代码的**完全**版本

```ini
distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists
distributionUrl=https\://services.gradle.org/distributions/gradle-4.4-all.zip
```

#### 怎么执行 gradlew 就一直卡在哪里？

嗯，这个问题，其实很常见。主要原因是你的上网姿势不对。解决方案呢有两个，要么你采用正确的上网姿势，做到科学上网；要么你采用一个国内 Gradle 镜像站点。如果采用镜像站，或者干脆你到官网下载下来 zip 包，架设到公司内网的某台服务器，这种情况下，你就需要用到有一个参数 `--gradle-distribution-url` ，当然这种情况你就不用写 `--gradle-version` 和 `--distribution-type` 了，因为 URL 里面都已经含有这些信息了。

```bash
gradle wrapper --gradle-distribution-url "https://local.dev/gradle-4.4-all.zip"
```

更多关于 wrapper 的知识可以去 <https://docs.gradle.org/current/userguide/gradle_wrapper.html>  查看。

### Gradle 工程概述

那么下面我们打开默认生成的 `build.gradle` 文件，将其改造成下面的样子：

```groovy
/*
 * 这个 build 文件是由 Gradle 的 `init` 任务生成的。
 *
 * 更多关于在 Gradle 中构建 Java 项目的信息可以查看 Gradle 用户文档中的
 * Java 项目快速启动章节
 * https://docs.gradle.org/4.0/userguide/tutorial_java_projects.html
 */
// 在这个区块中你可以声明你的 build 脚本需要的依赖和解析下载该依赖所使用的仓储位置
buildscript {
  ext {
    springBootVersion = '2.0.0.RELEASE'
  }
  repositories {
    jcenter()
  }
}
// 从 Spring Boot 2.0 起，你可以直接在 plugins 区块中声明 SpringBoot 插件了
plugins {
  id 'org.springframework.boot' version '2.0.0.RELEASE'
}
/*
 * 在这个区块中你可以声明使用哪些插件
 * apply plugin: 'java' 代表这是一个Java项目，需要使用 java 插件
 * 如果想生成一个 Intellij IDEA 的工程，类似的如果要生成
 * eclipse 工程，就写 apply plugin: 'eclipse'
 * 同样的我们要学的是 Spring Boot，所以应用 Spring Boot 插件
 */
apply plugin: 'java'
apply plugin: 'idea'
apply plugin: 'eclipse'
apply plugin: 'io.spring.dependency-management'

// 在这个区块中你可以声明编译后的 Jar 文件信息
jar {
  baseName = 'gtm-backend'
  version = '0.0.1-SNAPSHOT'
}

// 在这个区块中你可以声明在哪里可以找到你的项目依赖
repositories {
    // 使用 'jcenter' 作为中心仓储查询解析你的项目依赖。
    // 你可以声明任何 Maven/Ivy/file 类型的依赖类库仓储位置
    // 如果遇到下载速度慢的话可以换成阿里镜像
    // maven {url "http://maven.aliyun.com/nexus/content/repositories/central/"}
    mavenCentral()
}

// 在这个区块中你可以声明源文件和目标编译后的Java版本兼容性
sourceCompatibility = 1.8
targetCompatibility = 1.8

// 在这个区块你可以声明你的项目的开发和测试所需的依赖类库
dependencies {
  // 在 gradle 4.x 中 compile 已经 deprecated，请使用 implementation
  implementation('org.springframework.boot:spring-boot-starter-web')
  testImplementation('org.springframework.boot:spring-boot-starter-test')
}
```

 1. 提供 Gradle 脚本系统中的 Spring Boot 支持
 2. 简化执行和发布：它可以把所有 `classpath` 的类库构建成一个单独的可执行 jar 文件 （fat jar），这样可以简化你的执行和发布等操作或者按传统方式打包成 `war`
 3. 自动搜索入口文件：它会扫描 `public static void main()` 函数并且标记这个函数的宿主类为可执行入口。
 4. 简化依赖：一个典型的 `Spring` 应用还是需要很多依赖类库的，想要配置正确这些依赖挺麻烦的，所以这个插件提供了内建的依赖解析器会自动匹配和当前 `Spring Boot` 版本匹配的依赖库版本。

当我们在脚本中应用了 `io.spring.dependency-management` 这个插件时，插件会自动导入你当前要使用的 Spring Boot 版本的依赖清单，这个体验和 Spring Boot 在使用 Maven 时保持了高度一致。我们不必指定具体声明当依赖的版本号，比如:

```groovy
dependencies {
  implementation('org.springframework.boot:spring-boot-starter-web')
  testImplementation('org.springframework.boot:spring-boot-starter-test')
}
```

#### 引入 Spring Boot 依赖但不应用 Spring Boot 插件

有些时候，我们可能不希望创建一个 Spring Boot 工程，但希望引入 Spring Boot 的依赖。比如我们在进行一个类库工程的开发时，这个类库本身的工程其实并不可以直接启动，而是会被其他工程引用。这种情况下，我们就会希望只是单纯的引入 Spring Boot 的依赖。此时需要在 `build.gradle` 中的插件区块中使用 `apply false`

```groovy
plugins {
  id 'org.springframework.boot' version '{version}' apply false
}
```

然后应用 `dependency-management` 插件，并且在 `dependencyManagement` 区块中国导入来料清单中的依赖。

```groovy
apply plugin: 'io.spring.dependency-management'

dependencyManagement {
  imports {
    mavenBom org.springframework.boot.gradle.plugin.SpringBootPlugin.BOM_COORDINATES
  }
}
```

## 通过 Maven 创建

Maven 和 gradle 其实非常类似，只不过 Maven 采用了 `xml` 格式的配置文件，这也是由于 Maven 诞生时，`xml` 确实也还是被广泛认为是可读性很好的一种文件格式。

### 安装 Maven

和 Gradle 类似的，这一步只有第一次当系统中没有 Maven 的时候才需要，以后的项目创建并不需要这一个步骤。

* Linux: 在 `terminal` 中敲 `sdk install maven` 。假如你没有安装 `sdkman` 的话，请参考第一章内容进行安装。
* macOS：在 `terminal` 中敲 `brew install maven` ，或者你也可以按 Linux 的安装方式进行
* Windows：在命令行下执行 `scoop install maven` ，如果你没有安装 `scoop` ，还是请参考第一章相关内容

要确认是否安装成功，可以在开一个命令行窗口，敲 `mvn -v`，如果你可以看到类似的输出，那么就恭喜你了，安装成功！

```bash
Apache Maven 3.5.2 (138edd61fd100ec658bfa2d307c43b76940a5d7d; 2017-10-18T15:58:13+08:00)
Maven home: /usr/local/Cellar/maven/3.5.2/libexec
Java version: 1.8.0_111, vendor: Oracle Corporation
Java home: /Library/Java/JavaVirtualMachines/jdk1.8.0_111.jdk/Contents/Home/jre
Default locale: zh_CN, platform encoding: UTF-8
OS name: "mac os x", version: "10.13.3", arch: "x86_64", family: "mac"
```

### 创建 Maven 工程

使用 `mvn` 命令创建一个新的工程，区别与 Gradle 的是，我们是在上一级目录执行下面的命令，也就是说工程目录会在执行后被创建出来。

```bash
mvn -B archetype:generate \
  -DarchetypeGroupId=org.apache.maven.archetypes \
  -DgroupId=dev.local.gtm \
  -DartifactId=gtm-backend
```

这个命令帮我们建立一个一个使用 maven 进行管理的模版工程，下面的列表中给出了这个命令为我们创建了哪些文件和文件夹，以及它们的作用：

* `pom.xml`：用于组织工程结构和项目依赖的 `xml` 配置文件
* `src` 目录：用于存放源文件，如下图所示

![Maven 工程的源文件目录结构](/assets/2018-04-01-14-29-31.png)

而一个典型的 `pom.xml` 文件如下

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>dev.local.gtm</groupId>
  <artifactId>gtm-backend</artifactId>
  <packaging>jar</packaging>
  <version>1.0-SNAPSHOT</version>
  <name>my-app</name>
  <url>http://maven.apache.org</url>
  <dependencies>
    <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>3.8.1</version>
      <scope>test</scope>
    </dependency>
  </dependencies>
</project>
```

加入 Spring Boot 的支持只需要稍作改动即可：

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>dev.local.gtm</groupId>
  <artifactId>gtm-backend</artifactId>
  <packaging>jar</packaging>
  <version>1.0-SNAPSHOT</version>
  <name>my-app</name>
  <url>http://maven.apache.org</url>
  <parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.0.0.RELEASE</version>
  </parent>
  <dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
  </dependencies>
  <build>
    <plugins>
      <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
      </plugin>
    </plugins>
  </build>
</project>
```

Maven 也有 Wrapper 的概念，要生成 wrapper 的话，在工程目录下执行下面的命令

```bash
mvn -N io.takari:maven:wrapper
```

和 Gradle 类似的，会生成 `mvnw` ，`mvnw.cmd` 以及一个 `.mvn` 目录

* `mvnw`：用于 `*nix` 环境下的 maven wrapper 文件。
* `mvnw.cmd`：用于 `Windows` 环境下的 maven wrapper 文件

使用下面命令验证一下是否成功

```bash
./mvnw spring-boot:run
```

如果你看到类似下面的输出，那就说明一切正常

```bash
[INFO] Scanning for projects...
[INFO]
[INFO] ------------------------------------------------------------------------
[INFO] Building gtm-backend 1.0-SNAPSHOT
[INFO] ------------------------------------------------------------------------
[INFO]
[INFO] >>> spring-boot-maven-plugin:2.0.0.RELEASE:run (default-cli) > test-compile @ gtm-backend >>>
[INFO]
[INFO] --- maven-resources-plugin:3.0.1:resources (default-resources) @ gtm-backend ---
[INFO] Using 'UTF-8' encoding to copy filtered resources.
[INFO] skip non existing resourceDirectory gtm-backend/src/main/resources
[INFO] skip non existing resourceDirectory gtm-backend/src/main/resources
[INFO]
[INFO] --- maven-compiler-plugin:3.7.0:compile (default-compile) @ gtm-backend ---
[INFO] Nothing to compile - all classes are up to date
[INFO]
[INFO] --- maven-resources-plugin:3.0.1:testResources (default-testResources) @ gtm-backend ---
[INFO] Using 'UTF-8' encoding to copy filtered resources.
[INFO] skip non existing resourceDirectory gtm-backend/src/test/resources
[INFO]
[INFO] --- maven-compiler-plugin:3.7.0:testCompile (default-testCompile) @ gtm-backend ---
[INFO] Nothing to compile - all classes are up to date
[INFO]
[INFO] <<< spring-boot-maven-plugin:2.0.0.RELEASE:run (default-cli) < test-compile @ gtm-backend <<<
[INFO]
[INFO]
[INFO] --- spring-boot-maven-plugin:2.0.0.RELEASE:run (default-cli) @ gtm-backend ---
Hello World!
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 3.733 s
[INFO] Finished at: 2018-04-01T14:49:31+08:00
[INFO] Final Memory: 21M/227M
[INFO] ------------------------------------------------------------------------

```

## 通过 IDE 创建

在 Intellij IDEA 中创建 Spring Boot 工程只需要选择左侧的 `Spring Initializr` 作为工程模版

![在 IDEA 中创建 Spring Boot 工程](/assets/2018-04-01-14-55-23.png)

点击 Next 在接下来的向导页面中填写参数

![填写项目参数](/assets/2018-04-01-14-56-55.png)

然后选择项目的依赖

![选择 Spring Boot 项目的依赖](/assets/2018-04-01-14-58-47.png)

最后填写一下项目位置的有关信息就可以完成了

![最后一步](/assets/2018-04-01-15-00-50.png)

## 工程项目的组织

在大型软件开发中，一个项目解决所有问题显然不可取的，因为存在太多的开发团队共同协作，所以对项目进行拆分，形成多个子项目的形式是普遍存在的。而且一个子项目只专注自己的逻辑也易于维护和拓展，现在随着容器和微服务的理念逐渐获得大家的认可，多个子项目分别发布到容器和形成多个微服务也逐渐成为趋势。

我们的项目使用 `Gradle` 来处理多项目的构建，包括大项目和子项目的依赖管理以及容器的建立等。对于 `Gradle` 项目来说，我们会有一个根项目，这个根项目下会建立若干子项目，具体文件结构如下：

```bash
|--backend （根项目）
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

```groovy
include 'common'
include 'api'
include 'report'
rootProject.name = 'gtm-backend'
```

这样 `gtm` 就成为了根项目，而 `common` 、`api` 和 `report` 就是其之下的子项目。接下来，我们看一下根项目的 `build.gradle`，对于多项目构建来说，根项目的 `build.gradle` 中应该尽可能的配置各子项目中共同的配置，从而让子项目只配置自己不同的东西。

```groovy
// 一个典型的根项目的构建文件结构
buildscript {
    /*
     * 构建脚本区块可以配置整个项目需要的插件，构建过程中的依赖以及依赖类库的版本号等
     */
}

allprojects {
    /*
     * 在这个区块中你可以声明对于所有项目（含根项目）都适用的配置，比如依赖性的仓储等
     */
}

subprojects {
    /*
     * 在这个区块中你可以声明适用于各子项目的配置（不包括根项目哦）
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

其中，`buildscript` 区块用于配置 `gradle` 脚本生成时需要的东西，比如配置整个项目需要的插件，构建过程中的依赖以及在其他部分需要引用的依赖类库的版本号等，就像下面这样，我们在 `ext` 中定义了一些变量来集中配置了所有依赖的版本号，无论是根项目还是子项目都可以使用这些变量来指定版本号。这样做的好处是当依赖的版本更新时，我们无需四处更改散落在各处的版本号。此外在这个区块中我们还提供了项目所需的第三方 `Gradle` 插件所需的依赖：`spring-boot-gradle-plugin` 和 `dependency-management-plugin`，这样在后面，各子项目可以简单的使用诸如 `apply plugin: 'io.spring.dependency-management'`  等即可。

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
    }
}
```

`allprojects` 中可以声明对于所有项目（含根项目）都适用的配置，比如依赖性的仓储等。而 `subprojects` 和 `allprojects` 的区别在于 `subprojecrts` 只应用到子项目，而非根项目。所以大部分通用型配置可以通过 `subprojects` 和 `allprojects` 来完成。下面列出的样例配置中，我们为所有的项目包括根项目配置了依赖仓储以及软件的 `group`，同时为每个**子项目**配置了 `java` 和 `idea` 两个插件、版本号和通用的测试依赖。

```groovy
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

除此之外呢，为了展示一下 `project` 的用法， 我们这个例子里把每个子项目的依赖放到根 `build.gradle` 中的 `project(':子项目名')` 中列出，这样做有好处也有缺点，好处是依赖性的管理统一在根 `build.gradle` 完成，对于依赖的情况一目了然。当然缺点是每个项目更改依赖时都会造成根 `gradle` 的更新，这样的话如果一个项目有非常多的子项目时，会在协作上出现一些问题。所以请根据具体情况决定把依赖放到根 `build.gradle` 中的 `project(':子项目名')` 中还是放到各子项目的 `build.gradle` 中。

```groovy
project(':common') {
    dependencies {
        implementation("org.springframework.boot:spring-boot-starter-data-rest")
        implementation("org.springframework.boot:spring-boot-starter-data-mongodb")
        implementation("org.projectlombok:lombok:${lombokVersion}")
    }
}

project(':api') {
    dependencies {
        implementation project(':common')
        implementation("org.springframework.boot:spring-boot-devtools")
        implementation("org.springframework.boot:spring-boot-starter-security")
        implementation("io.jsonwebtoken:jjwt:${jjwtVersion}")
        implementation("org.projectlombok:lombok:${lombokVersion}")
    }
}

project(':report') {
    dependencies {
        implementation project(':common')
        // the following 5 are required by jasperreport rendering
        implementation files(["lib/simsun.jar"])
        implementation("org.springframework.boot:spring-boot-devtools")
        implementation("org.springframework.boot:spring-boot-starter-web")
        implementation("org.springframework:spring-context-support:${springCtxSupportVersion}")
        implementation("net.sf.jasperreports:jasperreports:${jasperVersion}")
        implementation("com.lowagie:itext:${itextVersion}")
        implementation("org.apache.poi:poi:${poiVersion}")
        implementation("org.olap4j:olap4j:${olap4jVersion}")
    }
}
```
