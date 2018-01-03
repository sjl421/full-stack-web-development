# Angular Material 介绍

Angular Material 是 Angular 团队官方开发的一套符合 Google Material 风格的 Angular UI 组件库。这套组件的使用方式上常常让我联想起 Android 的开发，个人感觉这应该也是 Google 努力的方向之一吧 -- 让 web 开发更像 app 开发或者后端开发。

安装 `@angular/material` 可以在 `terminal` 中敲入

```bash
yarn add @angular/material @angular/cdk
```

如果看到类似下面的输出，那就安装成功了

```bash
•100% ➜ yarn add @angular/material @angular/cdk
yarn add v1.3.2
[1/4] 🔍  Resolving packages...
[2/4] 🚚  Fetching packages...
[3/4] 🔗  Linking dependencies...
[4/4] 📃  Building fresh packages...
success Saved lockfile.
success Saved 2 new dependencies.
├─ @angular/cdk@5.0.2
└─ @angular/material@5.0.2
✨  Done in 29.78s.
```

## 组件类别

`@angular/material` 在 `2.0.0-beta.8` 之前是单独的一个 `package`，但后来团队把其中的一些公用功能以及组件抽离出来放到了一个单独的 `@angular/cdk` 包中。这个 `cdk` \( Component Dev Kit \) 以后可以作为你开发自己风格组件库的基础，因为它封装了很多公共特性的支持，你不需要从零开始。 `cdk` 中提供的主要功能如下图所示：

![Angular CDK 包含的内容](/assets/chap_2_2_001.png)

总体来说，`@angular/material` 提供了 30 多个组件以及主题和字体的支持，并通过 `@angular/flex-layout` 提供了 `flexbox` 布局的 angular 封装。

* 组件

  * 表单控件类：`Autocomplete` 、`Checkbox` 、`Datepicker` 、 `Form Field` 、 `Input` 、`Radio Button` 、`Select` 、`Slider` 、 `Slide Toggle`

  * 导航类：`Menu` 、`Sidenav` 、`Toolbar`

  * 布局类：`List` 、`Grid List` 、`Card` 、 `Stepper` 、 `Tabs` 、`Expansion Panel`

  * 按钮和提示类：`Button` 、`Button Toggle` 、`Icon` 、 `Progress Spinner` 、 `Progress Bar`

  * 弹出类：`Dialog` 、`Tooltip` 、`Snackbar` 、

  * 数据表格类：`Table` 、`Sort Header` 、`Paginator`

* 布局支持：通过独立的 `@angular/flex-layout` 提供，这个软件包不仅提供 flex 的封装，也提供响应式页面设计需要的各种 API 和指令。

* 主题支持：主题的支持主要由框架提供的一系列 `scss` 函数来实现，因此如果希望有主题的自定义时，需要以 `scss` 形式提供样式。

## 布局控件 - Sidenav

Google 的 Material 设计语言中，对于一个应用的布局经常采用的形式是一个内容区块加上一个侧面菜单，这个菜单既可以是滑出的也可以是固定在侧面的。

![常见的布局方式](/assets/chap_2_2_002.png)

所以呢，对于这种常见的布局，Google 提供了一种布局控件 -- `Sidenav` -- 来帮助开发者很方便的实现。

```html
<mat-sidenav-container fullscreen>
  <mat-sidenav #sidenav mode="over">
    <app-sidebar></app-sidebar>
  </mat-sidenav>
  <div class="site">
    <header>
      ...
    </header>
    <main>
      <router-outlet></router-outlet>
    </main>
    <footer>
      ...
    </footer>
  </div>
</mat-sidenav-container>
```

如上面的代码所示，这种布局一般需要两个组件相互配合，第一个是 `<mat-sidenav-container>`，这一个作为侧滑控件的容器，一般也可以用作整个 App 的容器。然后在这个容器中使用 `<mat-sidenav>` 构建可侧滑的内容。

其实还有一个 `<mat-sidenav-content>`，也就是我们可以把 `<sidenav>` 对应的内容可以放入这个 `<mat-sidenav-content>` 之中，但由于如果不写的话，Angular 也会默认创建一个元素把其他部分封装在里面，所以不写这个也是可以的。

这个组件需要封装在了一个叫做 `MatSidenavModule` 的模块中，所以如果要使用的话，需要导入这个模块，我们现在把它放到共享模块当中去：

```js
import { NgModule } from '@angular/core';
import { CommonModule } from '@angular/common';
import { MatSidenavModule } from '@angular/material';

const MATERIAL_MODULES = [
  MatSidenavModule,
];

const MODULES = [
  ...MATERIAL_MODULES,
  CommonModule,
];

@NgModule({
  declarations: [],
  imports: MODULES,
  exports: [
    ...MODULES,
  ]
})
export class SharedModule {}
```

这个模块除了提供了 `Sidenav` 之外，还提供了一个类似功能的组件 `Drawer` ，那么问题来了，这个 `Drawer` 和 `Sidenav` 的区别在哪儿呢？答案是区域的大小，如果是整个页面级的侧滑我们使用 `Sidenav` 而对于页面上的某个小区域的话，如果也要实现类似的效果，那么就使用 `Drawer`

```html
<mat-drawer-container class="container">
  <mat-drawer mode="side" opened="true">抽屉内容</mat-drawer>
  <mat-drawer-content>主要内容</mat-drawer-content>
</mat-drawer-container>
```

如果你看的足够仔细的话，会发现这个组件有一系列的属性，这些属性具体可以去 [https://material.angular.io/components/sidenav/overview](https://material.angular.io/components/sidenav/overview) 查阅官方文档。这里我们介绍几个最常用的属性，第一个是 `mode`

| mode | 说明 |
| :--- | :--- |
| over | 这个值是默认值，效果是侧边浮在主要内容之上 |
| push | 侧边会向右或向左挤走主要内容的部分区域 |
| side | 侧边回合主要内容并列 |

当然如果是抽屉的话，自然会有打开和关闭的状态，针对这两种状态，组件提供了若干方法、属性和事件：

| 名称 | 类型 | 描述 | 示例 |
| :--- | :--- | :--- | :--- |
| open | 方法 | 打开侧边 | sidenav.open\(\) |
| close | 方法 | 关闭侧边 | sidenav.close\(\) |
| toggle | 方法 | 反转当前状态 | sidenav.toggle\(\) |
| opened | 属性 | 打开的状态 | \[opened\]="status" |
| closed | 属性 | 关闭的状态 | \[closed\]="status" |
| closedStart | 事件 | 开始关闭的事件 | \(closedStart\)="handleClose\(\)" |
| openedStart | 事件 | 开始打开的事件 | \(openedStart\)="handleOpen\(\)" |

这个给出一个 `toggle` 的例子，我们给 `Sidenav` 起一个引用名字 `sidenav` ，然后在 `button` 的点击事件处理中调用 `toggle()`

```html
<mat-sidenav-container fullscreen>
  <mat-sidenav #sidenav mode="over">
    <app-sidebar></app-sidebar>
  </mat-sidenav>
  <div class="site">
    <header>
      <button (click)="sidenav.toggle()"> 切换开关状态 </button>
    </header>
    <main>
      <router-outlet></router-outlet>
    </main>
    <footer>
      ...
    </footer>
  </div>
</mat-sidenav-container>
```

## Flex 布局和 Angular Flex-layout

在 `flex` 布局之前，想使用 `HTML/CSS/JS` 去进行一些特殊布局的话，一般都需要进行一些 hack。可以说 `css` 引入 `flex` 布局给前端开发注入了一股清流。当然之后还有 `grid` 布局等现代布局方式，但由于支持 `grid` 布局的浏览器还不是很普遍，而各主流浏览器对 `flex` 的支持则已经比较成熟了。我们这里同样不会对 `flex` 做详细介绍，有需要了解的童鞋可以去 [https://css-tricks.com/snippets/css/a-guide-to-flexbox/](https://css-tricks.com/snippets/css/a-guide-to-flexbox/) 去学习相关知识。

而 `@angular/flex-layout` 是 Angular 团队给出一个基于 `flex` 布局的 Angular 类库，那么为什么不直接使用 `flex` 而要使用这个封装类库呢？其实答案并非非此即彼，很多时候我们是可以混用的，但 `@angular/flex-layout` 提供了方便的指令用来自动化 `flex` 和媒体查询，而且其最大的优势是提供了一个强大的 `Responsive API` ，让开发者可以方便的开发适合多种屏幕布局的响应式应用。

首先安装 `@angular/flex-layout`

```bash
yarn add @angular/flex-layout
```

### 常见的指令和用法

* fxLayout：标识一个元素为 `flex` 容器，值分为 `row` 和 `column` ，指明容器方向。
* fxLayoutAlign：指定子元素按容器方向和交叉轴 \(`cross axis` \) 的排布方式，相当于 `css` 中的 `justify-content` 和 `align-content`
* fxFlex：相当于 `css` 中的 `flex` ，可以接受三个值 -- `flex-grow` 、 `flex-shrink` 、 `flex-basis`

这些指令可以和 `@angular/flex-layout` 的媒体查询断点结合使用

| 断点 | 媒体查询 |
| :--- | :--- |
| xs | 'screen and \(max-width: 599px\)' |
| sm | 'screen and \(min-width: 600px\) and \(max-width: 959px\)' |
| md | 'screen and \(min-width: 960px\) and \(max-width: 1279px\)' |
| lg | 'screen and \(min-width: 1280px\) and \(max-width: 1919px\)' |
| xl | 'screen and \(min-width: 1920px\) and \(max-width: 5000px\)' |
| lt-sm | 'screen and \(max-width: 599px\)' |
| lt-md | 'screen and \(max-width: 959px\)' |
| lt-lg | 'screen and \(max-width: 1279px\)' |
| lt-xl | 'screen and \(max-width: 1919px\)' |
| gt-xs | 'screen and \(min-width: 600px\)' |
| gt-sm | 'screen and \(min-width: 960px\)' |
| gt-md | 'screen and \(min-width: 1280px\)' |
| gt-lg | 'screen and \(min-width: 1920px\)' |

也就是我们可以这样来写一个响应式布局

```html
<div fxFlex="50%" fxFlex.gt-sm="100%">
...
</div>
```

### 完成首页布局

我们要对首页进行的布局初看上去比较简单，但当内容较少的时候，`footer` 一般会上移，这就比较难看了，我们希望的是，无论内容多少，`footer` 始终在页尾，下面我们看看怎样使用 `flex` 布局达成这个效果：

```css
// styles.scss

html, body, app-root, mat-sidenav-container {
  margin: 0;
  width: 100%;
  height: 100%;
}

.site {
  width: 100%;
  min-height: 100%;
}

.full-width {
  width: 100%;
}

.fill-remaining-space {
  // 使用 flexbox 填充剩余空间
  // @angular/material 中的很多控件使用了 flex 布局
  flex: 1 1 auto;
}
```

首先我们在 `styles.scss` 中将几个顶级元素 \( `html` `body` `app-root` 以及 `mat-sidenav-container`  \) 的边距设为 `0` ，并且让它们充满整个空间。然后我们需要把主要内容区域设置成一个垂直方向的  `flex box` \( `<div class="site" fxLayout="column">` \) ，这样它的子元素 \( `header` 、 `main` 、 `footer`  \) 会按照纵向排列。而我们对于 `main` 又设置了 `fxFlex="1"` 使得这个元素会尽可能占据剩余空间，这样它就会把 `header` 和 `footer` 分别挤到页首和页尾，这样也就达成了我们的目的。

```html
<mat-drawer-container fullscreen>
  <mat-drawer #sidenav mode="over">
    ...
  </mat-drawer>
  <div class="site" fxLayout="column">
    <header>
      ...
    </header>
    <main fxFlex="1" fxLayout="column" fxLayoutAlign="center strech">
      <router-outlet></router-outlet>
    </main>
    <footer>
      ...
    </footer>
  </div>
</mat-drawer-container>
```



