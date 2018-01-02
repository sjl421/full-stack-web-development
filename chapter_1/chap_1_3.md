# 工程项目的结构

## 前端项目

### 安装 Angular CLI

`Angular CLI` 是一套命令行工具，可以生成工程的脚手架，它隐藏了很多配置的繁琐细节，可以让我们更专注在逻辑代码的实现上。我们可以通过 `npm` 或者 `yarn` 来安装:

```bash
npm install -g @angular/cli
```

或者

```bash
yarn add global @angular/cli
```

以上两种安装方式是个人推荐的安装方式，但是由于国内网络的限制，有时候可能安装时间过长或者有些软件包无法下载，这个时候需要你掌握科学上网的姿势。如果实在无法安装成功的话，也可以采用淘宝团队提供的 `cnpm` ，这个 `cnpm` 可以理解成使用淘宝的镜像软件仓库的 `npm` 中国加速版。

```bash
npm install -g cnpm --registry=https://registry.npm.taobao.org
```

安装 `cnpm` 之后，可以使用 `cnpm` 替代文中使用 `yarn` 或 `npm` 进行安装的命令。

可以通过如下命令测试 `cli` 是否安装成功，这个 `ng` 命令我们会经常用到，为什么叫 `ng` ？因为 `angular` 的简写就是 `ng` 啦：

```bash
ng version
```

如果输出的是类似下面的样子，那么就一切 OK 啦

        _                      _                 ____ _     ___
       / \   _ __   __ _ _   _| | __ _ _ __     / ___| |   |_ _|
      / △ \ | '_ \ / _` | | | | |/ _` | '__|   | |   | |    | |
     / ___ \| | | | (_| | |_| | | (_| | |      | |___| |___ | |
    /_/   \_\_| |_|\__, |\__,_|_|\__,_|_|       \____|_____|___|
                   |___/

    Angular CLI: 1.6.0
    Node: 8.9.0
    OS: darwin x64
    Angular:
    ...

如果不是呢，那么很不幸，就是安装过程中有错误，你需要重复上面的安装步骤直至安装成功为止。

### 搭建前端项目框架

首先我们使用 `Angular CLI` 创建一个新的工程：

```bash
ng new client --style scss --skip-install
```

这里注意 `ng new <project name>` 就是创建一个 `Angular` 工程，而后面的参数 `--style scss` 是告诉 `cli` 在创建工程时，我们会采用 `scss` 作为工程的样式工具，如果不加这个参数的话，工程默认的是 `css` 。`scss` 我们这里不会详细的讲解，有疑问的同学可以去 [http://sass-lang.com/](http://sass-lang.com/) 学习，简单来说，`scss` 就是可编程的 `css` 。

这个命令执行的过程可能会有点长，而且它默认的使用了 `yarn` 进行安装，所以我们加了 `--skip-install` 这个参数跳过依赖的安装，这样后面如果你想使用 `cnpm` 可以进入 `client` 目录手动安装依赖 `cnpm install` ，当然使用推荐的安装命令的话就是 `yarn install` 或 `npm install`。需要注意的一点是 `client` 目录才是工程目录，大部分的 `cli` 子命令都需要在此目录下执行才能生效。

```bash
> ng new client --style scss --skip-install
  create client/README.md (1022 bytes)
  create client/.angular-cli.json (1243 bytes)
  create client/.editorconfig (245 bytes)
  create client/.gitignore (516 bytes)
  create client/src/assets/.gitkeep (0 bytes)
  create client/src/environments/environment.prod.ts (51 bytes)
  create client/src/environments/environment.ts (387 bytes)
  create client/src/favicon.ico (5430 bytes)
  create client/src/index.html (293 bytes)
  create client/src/main.ts (370 bytes)
  create client/src/polyfills.ts (2405 bytes)
  create client/src/styles.scss (80 bytes)
  create client/src/test.ts (1085 bytes)
  create client/src/tsconfig.app.json (211 bytes)
  create client/src/tsconfig.spec.json (304 bytes)
  create client/src/typings.d.ts (104 bytes)
  create client/e2e/app.e2e-spec.ts (288 bytes)
  create client/e2e/app.po.ts (208 bytes)
  create client/e2e/tsconfig.e2e.json (235 bytes)
  create client/karma.conf.js (923 bytes)
  create client/package.json (1311 bytes)
  create client/protractor.conf.js (722 bytes)
  create client/tsconfig.json (363 bytes)
  create client/tslint.json (3040 bytes)
  create client/src/app/app.module.ts (316 bytes)
  create client/src/app/app.component.scss (0 bytes)
  create client/src/app/app.component.html (1141 bytes)
  create client/src/app/app.component.spec.ts (986 bytes)
  create client/src/app/app.component.ts (208 bytes)
Successfully initialized git.
Project 'client' successfully created.
```

如果看到类似上面的输出结果，工程就生成完毕了，我们进入 `client` 目录。

```bash
cd client
```

通过 `yarn install` 或 `npm install` 安装依赖，如果成功，输出的结果和下面的类似：

```bash
> yarn install
yarn install v1.3.2
info No lockfile found.
[1/4] 🔍  Resolving packages...
[2/4] 🚚  Fetching packages...
[3/4] 🔗  Linking dependencies...
warning "@angular/cli > @angular-devkit/schematics > @schematics/schematics@0.0.11" has unmet peer dependency "@angular-devkit/core@0.0.22".
[4/4] 📃  Building fresh packages...
success Saved lockfile.
✨  Done in 70.30s.
```

我们的工程最后会如下图的组织形式一样

![](/assets/chap_1_3_client_structure.png)

但在一开始，我们先建立两个模块，一个叫核心模块 \(CoreModule\) ，另一个叫共享模块 \(SharedModule\)。核心模块的作用是初始化应用以及加载必要需要**单例**的功能，最常见的情况是，我们通常把 Http 的服务放到核心模块中，因为通常情况下，我们希望服务只被创建一次。

```bash
> ng g m core
  create src/app/core/core.module.ts (188 bytes)
```

上面这个简单命令就是创建模块的命令，`ng` 后的那个 `g` 是 `generate` 的缩写，也就是生成的意思。而 `m` 自然就是  `module` 的缩写，就是模块的意思。如果你想生成组件，类似的就使用 `ng g c <component name>` ，生成指令就写成 `ng g d <directive name>` ，具体的参数其实很多，大家可以使用 `ng help <subcommand name>` 的形式查看。

回到我们的核心模块，将其改造成下面的样子

```js
import { NgModule, Optional, SkipSelf } from '@angular/core';

export class CoreModule {

  constructor(
    @Optional() @SkipSelf() parentModule: CoreModule) {
    if (parentModule) {
      throw new Error('CoreModule 已经装载，请仅在 AppModule 中引入该模块。');
    }
  }
}
```

这里改造的目的是让核心模块变成一个单例：应用生命周期内只加载一次。具体的逻辑如果看不懂，请转到第二章的依赖性注入那一节去学习。

而共享模块通常包含需要在应用的多处使用的组件、指令、管道或其他共享代码，在共享模块中，我们经常会做模块的导入和导出，这个看似吃力不讨好的操作是为了让所有导入共享模块的其他功能模块不用再去导入。

```bash
ng g m shared
  create src/app/shared/shared.module.ts (190 bytes)
```

同样的，我们需要改造一下这个共享模块，为什么要把要导入的，导出的都写成一个数组？这是为了后期导入、导出的东东很多的时候，我们可以方便的重用。

```js
import { NgModule } from '@angular/core';
import { CommonModule } from '@angular/common';

const MODULES = [
  CommonModule,
];

const DECLARATIONS = [];
const EXPORT_COMPONENTS = [];
const ENTRYCOMPONENTS = [];

@NgModule({
  imports: MODULES,
  exports: [
    ...MODULES,
    ...EXPORT_COMPONENTS
  ],
  declarations: DECLARATIONS,
  entryComponents: [
    ENTRYCOMPONENTS
  ]
})
export class SharedModule {
}
```

## 后端项目

### 多项目构建

在大型软件开发中，一个项目解决所有问题显然不可取的，因为存在太多的开发团队共同协作，所以对项目进行拆分，形成多个子项目的形式是普遍存在的。而且一个子项目只专注自己的逻辑也易于维护和拓展，现在随着容器和微服务的理念逐渐获得大家的认可，多个子项目分别发布到容器和形成多个微服务也逐渐成为趋势。

我们的项目使用 `Gradle` 来处理多项目的构建，包括大项目和子项目的依赖管理以及容器的建立等。对于 `Gradle` 项目来说，我们会有一个根项目，这个根项目下会建立若干子项目，具体文件结构如下：

```bash
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

```gradle
include 'common'
include 'api'
include 'report'
rootProject.name = 'spring-boot-tut'
```

这样 `spring-boot-tut` 就成为了根项目，而 `common` 、`api` 和 `report` 就是其之下的子项目。接下来，我们看一下根项目的 `build.gradle`，对于多项目构建来说，根项目的 `build.gradle` 中应该尽可能的配置各子项目中共同的配置，从而让子项目只配置自己不同的东西。

```gradle
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

```gradle
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

```gradle
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

```gradle
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



