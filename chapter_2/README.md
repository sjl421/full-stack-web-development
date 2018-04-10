# 使用 Angular 快速构造前端原型

本章会从 Angular 的核心概念出发，第一节以一系列小例子阐释这些概念的意义和使用方法。如果有 Angular 基础的同学可以跳过或者摘选自己感兴趣的内容看。在第二节中，我们会一起来认识 Angular 的官方 UI 组件库 -- Angular Material，这是一套遵循谷歌 Material Design 风格的组件库。使用它的好处在于可以在组件标准化、动画、兼容性方面节省很大精力，即使你不熟悉 CSS 也可以做出很好看的 UI 效果。我们在第二节会一起学习几个较常见的组件，当然只是最初的简单框架和页面，使用的是 Angular Material 组件库和 Angular FlexLayout 布局库。第三节我们会一起学习 Angular Material 的主题支持，学会如何定制化主题。第四节我们使用容器来构建我们的应用， 我们不会  专门的去讲关于容器的知识，但在  书中需要使用容器的地方会  有相应的说明。使用容器的原因是它可以  让整个开发部署的流程更加自动化、更有生产效率。

## 项目工程结构

前端采用的工程结构是类似下面的结构，第一层的几个文件夹

* docker -- 用于容器的构建文件，比如 Dockerfile 以及在构建 Docker 时需要的一些文件
* e2e -- 端到端测试文件，web 自动化集成测试，基于 protractor
* src -- 源文件目录，我们平时主要接触的是这个目录

其中第一层还包含若干文件

* angular.json -- Angular 工程配置文件
* docker-compose.yml -- 容器集成脚本
* karma.conf.js -- 自动化单元测试配置文件
* package.json -- 项目依赖文件
* protractor.conf.js -- 端到端测试配置文件
* README.md -- Markdown 格式的项目说明文档
* tsconfig.json -- typescript 的配置文件
* tslint.json -- typescript 的 lint 配置
* yarn.lock -- yarn 的依赖管理文件

```log
├── docker/
│   └── nginx/
│       ├── conf.d/
│       │   └── default.conf
│       └── Dockerfile
├── e2e/
│   ├── app.e2e-spec.ts
│   ├── app.po.ts
│   └── tsconfig.e2e.json
├── src/
├── angular.json
├── docker-compose.yml
├── karma.conf.js
├── package.json
├── protractor.conf.js
├── README.md
├── tsconfig.json
├── tslint.json
└── yarn.lock
```

`src` 目录下的文件结构是我们项目主要使用的，所以展开讲一下。

* `app` -- 具体应用的源代码目录， `src` 下面可以有多个应用。
  * 在 `app` 目录下，我们一般按 Angular 的模块进行目录的规划，也就是说一般是每个模块一个目录，根模块除外。
  * 一个标准的应用，我们为它建立两个比较特殊的模块 `CoreModule` 和 `SharedModule` 。前者每个应用启动时只初始化一次，后者的作用是共享一些常用模块和组件，避免在多个模块导入类似模块、组件、指令等的麻烦。
  * 所有的领域对象我们存储在 `domain` 文件夹。
  * 工具类我们放在 `utils` 文件夹。
* `assets` -- 静态资源文件目录，比如图片等。
* `environments` -- 环境配置目录，在此可以建立多个环境配置文件，比如用于生产环境的 `prod`，用于开发的 `dev` 或者其他。

```log
├── src/
│   ├── app/
│   │   ├── core/
│   │   │   ├── components/
│   │   │   │   ├── footer.component.ts
│   │   │   │   ├── header.component.ts
│   │   │   │   ├── page-not-found.component.spec.ts
│   │   │   │   ├── page-not-found.component.ts
│   │   │   │   └── sidebar.component.ts
│   │   │   ├── containers/
│   │   │   │   └── app/
│   │   │   │       ├── app.component.html
│   │   │   │       ├── app.component.scss
│   │   │   │       ├── app.component.spec.ts
│   │   │   │       └── app.component.ts
│   │   │   ├── app-routing.module.ts
│   │   │   ├── core.module.ts
│   │   │   └── material.module.ts
│   │   ├── domain/
│   │   │   ├── auth.ts
│   │   │   ├── menu.ts
│   │   │   ├── role.ts
│   │   │   └── user.ts
│   │   ├── profile/
│   │   │   ├── profile-routing.module.ts
│   │   │   └── profile.module.ts
│   │   ├── reducers/
│   │   │   └── index.ts
│   │   ├── shared/
│   │   │   ├── directives/
│   │   │   ├── permission.ts
│   │   │   └── shared.module.ts
│   │   ├── utils/
│   │   │   └── permission.util.ts
│   │   └── app.module.ts
│   ├── assets/
│   │   └── img/
│   │       └── 400_night_light.jpg
│   ├── environments/
│   │   ├── environment.prod.ts
│   │   └── environment.ts
│   ├── favicon.ico // 站点图标
│   ├── index.html // Angular 会编译成一个单页应用，这个就是那个单页了
│   ├── main.ts // 应用入口文件
│   ├── polyfills.ts // 浏览器兼容性适配文件
│   ├── styles.scss // 顶层样式文件
│   ├── test.ts // 测试入口文件
│   ├── theme.scss // 主题样式文件
│   ├── tsconfig.app.json // 开发时的 typescript 配置
│   ├── tsconfig.spec.json // 测试时的 typescript 配置
│   └── typings.d.ts
```
