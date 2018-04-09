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

这个模块除了提供了 `Sidenav` 之外，还提供了一个类似功能的组件 `Drawer` ，那么问题来了，这个 `Drawer` 和 `Sidenav` 的区别在哪儿呢？答案是区域的大小，如果是整个页面级（也就是全屏）的侧滑我们使用 `Sidenav` 而对于页面上的某个小区域的话，如果也要实现类似的效果，那么就使用 `Drawer`

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

而 `@angular/flex-layout` 是 Angular 团队给出一个基于 `flex` 布局的 Angular 类库，那么为什么不直接使用 `flex` 而要使用这个封装类库呢？其实答案并不是非此即彼的，很多时候我们是可以混用的，但 `@angular/flex-layout` 提供了方便的指令用来自动化 `flex` 和媒体查询，而且其最大的优势是提供了一个强大的 `Responsive API` ，让开发者可以方便的开发适合多种屏幕布局的响应式应用。

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

html, body, app-root {
  margin: 0;
  width: 100%;
  height: 100%;
}

.site {
  width: 100%;
  min-height: 100%;
}

.full-width {
  // 这个类和布局无关，后面会使用它来使某些组件可以撑满空间
  width: 100%;
}

.fill-remaining-space {
  // 使用 flexbox 填充剩余空间
  // @angular/material 中的很多控件使用了 flex 布局
  flex: 1 1 auto;
}
```

首先我们在 `styles.scss` 中将几个顶级元素 \( `html` `body` `app-root`  \) 的边距设为 `0` ，并且让它们充满整个空间。 `fullscreen` 指令起的作用和前面的 `css` 差不多，就是让 `mat-sidenav-container` 也变成边距为 `0` ，长宽 `100%` 。然后我们需要把主要内容区域设置成一个垂直方向的  `flex box` \( `<div class="site" fxLayout="column">` \) ，这样它的子元素 \( `header` 、 `main` 、 `footer`  \) 会按照纵向排列。而我们对于 `main` 又设置了 `fxFlex="1"` 使得这个元素会尽可能占据剩余空间，这样它就会把 `header` 和 `footer` 分别**挤**到页首和页尾，这样也就达成了我们的目的。

```html
<mat-sidenav-container fullscreen>
  <mat-sidenav #sidenav mode="over">
    ...
  </mat-sidenav>
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
</mat-sidenav-container>
```

## 封装 Header/Footer/Sidebar

接下来，我们会创建三个组件，分别渲染页面头部、尾部和侧边栏。

### Header （页首）

这个页首我们按照如下需求实现

1. 纯色背景色
2. 左侧有一个控制侧边栏显示的图标按钮，点击可以切换侧边栏的显示与隐藏
3. 右侧有一个普通主题和黑夜模式的切换控件
4. 最右侧有一个退出按钮
5. 这些按钮应该可以控制显示和隐藏，因为后面我们会应对登录前后的状态。比如登陆前侧边栏菜单是否显示，退出按钮不应显示等等。

我们会采用 `Angular Material` 的 `toolbar` ，工具栏一般适合有一排按钮或者多排按钮，也可以用作页首或页尾。

要使用 `Angular Material toolbar` 的话，你需要导入 `MatToolbarModule`。

```ts
import { MatToolbarModule } from '@angular/material/toolbar';
```

这个 Module 提供了两个组件： `MatToolbar` 和 `MatToolbarRow` 。前者当然就是定义一个 `toolbar` ，后面的这个是如果希望 `toolbar` 是多行的情况下，我们会采用类似下面的写法。此外 `MatToolbar` 提供了一个 `color` 属性，用来设置背景色，由于遵循 Material 标准，所以可选的值有 `primary` 、 `accent` 和 `warn` 。

```html
<mat-toolbar color="primary">
  <mat-toolbar-row>
    <span>第一行</span>
  </mat-toolbar-row>

  <mat-toolbar-row>
    <span>第二行</span>
  </mat-toolbar-row>
</mat-toolbar>
```

我们的项目中为了集中管理 Material 组件，我们可以创建一个新的模块 `MyMaterialModule`，把我们需要的 `Material` 组件所需要的 `module` 都统一做一次导出。

```ts
import { NgModule } from '@angular/core';
import {
  MatButtonModule,
  MatIconModule,
  MatListModule,
  MatMenuModule,
  MatSidenavModule,
  MatSlideToggleModule,
} from '@angular/material';
import { FlexLayoutModule } from '@angular/flex-layout';

/**
 * NgModule that includes all Material modules.
 */
@NgModule({
  exports: [
    MatButtonModule,
    MatIconModule,
    MatListModule,
    MatMenuModule,
    MatSidenavModule,
    MatSlideToggleModule,
    FlexLayoutModule,
  ]
})
export class MyMaterialModule {}
```

然后在 `CoreModule` 中导入这个模块，这样在全局我们只导入了一次 Material。

```ts
imports: [
  CommonModule,
  MyMaterialModule,
  HttpClientModule, //如果有 mat-icon 或 mat-icon-button 是必须要导入的
  BrowserAnimationsModule // material 的动画效果必须导入这个模块
]
```

接下来，我们要实现 `HeaderComponent`，这里我们把它设计成一个“笨组件”。所谓笨组件就是指这个组件并不知道业务逻辑，它的全部行为都由若干属性定义，也就是它就像一个木偶组件，外部使用时设置它做什么，它就做什么。这么笨的组件能有什么用呢？其实我们使用的大部分组件都是“笨组件”，比如 HTML 的 `<button>`、`<link>`、`<div>` 等等。由于“笨组件”不了解业务逻辑反倒给它更灵活的发挥空间，复用性也更好。一般来说，一个系统中“笨组件”更多，而“聪明组件”较少为好，这样需求发生变更时，系统修改起来会比较快。

对于 `HeaderComponent`，我们定义 2 个输入型属性和 3 个输出型属性（在外部看来是事件）

* title -- 站点名称，在页首以较大字号显示
* hideForGuest -- 是否对未登录客户隐藏
* toggleMenuEvent -- 点击左侧按钮
* toggleDarkModeEvent -- 切换黑夜模式
* logoutEvent -- 点击退出按钮

```ts
import {
  Component,
  ChangeDetectionStrategy,
  Input,
  Output,
  EventEmitter
} from '@angular/core';

@Component({
  selector: 'app-header',
  template: `
  <mat-toolbar color="primary">
    <button mat-icon-button (click)="toggleMenu()" *ngIf="!hideForGuest">
      <mat-icon>menu</mat-icon>
    </button>
    <span>{{ title }}</span>
    <span class="fill-remaining-space"></span>
    <mat-slide-toggle (change)="toggleDarkMode($event.checked)" *ngIf="!hideForGuest">黑夜模式</mat-slide-toggle>
    <button *ngIf="!hideForGuest" mat-button (click)="handleLogout()">退出</button>
  </mat-toolbar>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class HeaderComponent {
  @Input() title = '企业协作平台';
  @Input() hideForGuest = false;
  @Output() toggleMenuEvent = new EventEmitter<void>();
  @Output() toggleDarkModeEvent = new EventEmitter<boolean>();
  @Output() logoutEvent = new EventEmitter();

  toggleMenu() {
    this.toggleMenuEvent.emit();
  }

  handleLogout() {
    this.logoutEvent.emit();
  }

  toggleDarkMode(checked: boolean) {
    this.toggleDarkModeEvent.emit(checked);
  }
}
```

这个组件中我们除了用到了 `mat-toolbar` ，还用到了 `mat-icon-button` 、 `mat-buton` 、 `mat-icon` 和 `mat-slide-toggle` 。

#### 按钮

按钮这个组件应该是最常用的 Material 组件之一了，它有很多种类型。

指令 | 描述
---------|----------
 `mat-button` | 标准的长方形按钮，没有凸起的视觉效果
 `mat-raised-button` | 标准的长方形按钮，有凸起的视觉效果
 `mat-icon-button` | 图标按钮，透明背景的圆形按钮，一般会包含一个 `mat-icon`
 `mat-fab` | 浮动按钮，它是有凸起效果的圆形按钮，默认背景色为 `accent`
 `mat-mini-fab` | 和 `mat-fab` 类似，但是更小

需要注意的是，除了 `<button>` 可以应用这个指令，`<a>` 也可以。按钮提供了下列的属性和方法

属性/方法 | 描述
---------|----------
 `@Input() color: ThemePalette` | 颜色属性
 `@Input() disableRipple: boolean` | 禁用水波纹效果，默认是点击时有水波纹效果
 `@Input() disabled: boolean` | 是否禁用该按钮
 `ripple: MatRipple` | 设置按钮的水波纹实例
 `focus()` | 聚焦

#### 图标

`mat-icon` 顾名思义是一个图标组件，支持 Google 为 Material Design 设计的图标字体 `Material Icon Font` 。使用 `mat-icon` 需导入 `MatIconModule`

```ts
import { MatIconModule } from '@angular/material/icon';
```

这个字体中含有哪些图标可以去 <https://material.io/icons/> 中查看。在应用中使用图标字体有几个好处

1. 字体体积较小，比图片的体积小很多
2. 字体可以无损缩放，所以可以适配多种设备分辨率

当然除了支持自己的图标字体，它也支持矢量 `svg` 图标和其他字体图标，比如 `FontAwesome` <https://fontawesome.com/icons> 等。

我们首先来看第一种方式，和 `Material Icon Font` 的配合使用，我们将字体图标的名称嵌入一对闭合的 `mat-icon` 标签中，就可以显示图标了。

```html
<mat-icon>home</mat-icon>
```

由于 `FontAwesome` 字体是采用 `css` 中的 `:before` 选择器来显示图标的。要和类似 `FontAwesome` 字体配合的话，我们需要设置 `mat-icon` 中的 `fontSet` 属性为其 `css` 类名，然后在 `fontIcon` 属性设置其图标名称。比如说在 FontAwesome 中的 🎂 的 `HTML` 代码如果是下面的样子

```html
<i class="fas fa-birthday-cake"></i>
```

那么我们使用 `mat-icon` 可以写成

```html
<mat-icon fontSet="fa" fontIcon="fa-birthday-cake"></mat-icon>
```

当然也可以通过 `MatIconRegistry` 设置一个别名

```ts
constructor(iconRegistry: MatIconRegistry, sanitizer: DomSanitizer) {
    iconRegistry
        .registerFontClassAlias('fontawesome', 'fa');
  }
```

然后写成下面的样子

```html
<mat-icon fontSet="fontawesome" fontIcon="fa-birthday-cake"></mat-icon>
```

这个 `MatIconRegistry` 是由 `MatIconModule` 提供的一个服务，它用于注册和显示 `mat-icon` 使用的图标。几个常用的方法如下，更多信息可以查看 <https://material.angular.io/components/icon/api>

方法 | 描述
---------|----------
 `addSvgIcon(iconName: string, url: SafeResourceUrl)` | 通过 URL 注册一个图标
 `addSvgIconInNamespace(namespace: string, iconName: string, url: SafeResourceUrl)` | 使用指定的命名空间通过 URL 注册一个图标
 `addSvgIconSet(url: SafeResourceUrl)` | 使用默认命名控件通过 URL 注册一组图标
 `addSvgIconSetInNamespace(url: SafeResourceUrl)` | 使用指定的命名空间通过 URL 注册一组图标
 `registerFontClassAlias(alias: string, className: string)` | 为图标字体使用的 `css` 类名称定义一个别名

如果要使用 `svg` 作为图标的话，我们需要注入 `MatIconRegistry` 和 `DomSanitizer` ，然后将 `svg` 图标注册到 `MatIconRegistry`

```ts
constructor(iconRegistry: MatIconRegistry, sanitizer: DomSanitizer) {
    iconRegistry.addSvgIcon(
        'thumbs-up',
        sanitizer.bypassSecurityTrustResourceUrl('assets/img/examples/thumbup-icon.svg'));
  }
```

然后在模版中就可以通过设置 `mat-icon` 的 `svgIcon` 为刚才注册的名称，这样 `svg` 图标就会显示出来了。

```html
<mat-icon svgIcon="project"></mat-icon>
```

当然在很多时候，我们希望不是在某个组件加载的时候才逐一的加载图标，因为那样的话，我们每个使用图标的组件可能就都得注入重复的 `MatIconRegistry` 和 `DomSanitizer` 和重复的初始化代码。不仅代码冗余，而且性能很差。所以我们把 `svg` 的载入等操作放入 `CoreModule` 的构造函数中，这样就在应用启动后只初始化一次，而且以后添加或修改资源时，位置也很集中。其他组件都是单纯的消费者，无需关心这些 `svg` 或者字体图标的注册。

```ts
import { MatIconRegistry } from '@angular/material';
import { DomSanitizer } from '@angular/platform-browser';

/**
 * 加载图标，包括 svg 图标和 FontAwesome 字体图标等
 *
 * @param ir a MatIconRegistry 实例，用于注册图标资源
 * @param ds a DomSanitizer 实例，用于忽略安全检查返回一个 URL
 */
export const loadIconResources = (ir: MatIconRegistry, ds: DomSanitizer) => {
  const imgDir = 'assets/img';
  const avatarDir = `${imgDir}/avatar`;
  const sidebarDir = `${imgDir}/sidebar`;
  const iconDir = `${imgDir}/icons`;
  const dayDir = `${imgDir}/days`;
  ir
    .addSvgIconSetInNamespace(
      'avatars',
      ds.bypassSecurityTrustResourceUrl(`${avatarDir}/avatars.svg`)
    )
    .addSvgIcon(
      'unassigned',
      ds.bypassSecurityTrustResourceUrl(`${avatarDir}/unassigned.svg`)
    )
    .addSvgIcon(
      'project',
      ds.bypassSecurityTrustResourceUrl(`${sidebarDir}/project.svg`)
    )
    .addSvgIcon(
      'projects',
      ds.bypassSecurityTrustResourceUrl(`${sidebarDir}/projects.svg`)
    )
    .addSvgIcon(
      'month',
      ds.bypassSecurityTrustResourceUrl(`${sidebarDir}/month.svg`)
    )
    .addSvgIcon(
      'week',
      ds.bypassSecurityTrustResourceUrl(`${sidebarDir}/week.svg`)
    )
    .addSvgIcon(
      'day',
      ds.bypassSecurityTrustResourceUrl(`${sidebarDir}/day.svg`)
    )
    .addSvgIcon(
      'move',
      ds.bypassSecurityTrustResourceUrl(`${iconDir}/move.svg`)
    )
    .registerFontClassAlias('fontawesome', 'fa');
};
```

然后在 `CoreModule` 的构造函数中调用我们的工具函数：

```ts
// 略去注解和导入部分
export class CoreModule {
  constructor(
    @Optional()
    @SkipSelf()
    parentModule: CoreModule,
    ir: MatIconRegistry,
    ds: DomSanitizer
  ) {
    if (parentModule) {
      throw new Error('CoreModule 已经装载，请仅在 AppModule 中引入该模块。');
    }
    // 加载我们刚刚创建的工具函数
    loadIconResources(ir, ds);
  }
}
```

当然为了支持这些图标字体，我们还需要引入一些 `css`，请将 `src/index.html` 更改成下面的样子

```html
<!doctype html>
<html lang="en">

<head>
  <meta charset="utf-8">
  <title>Gtm</title>
  <base href="/">

  <meta name="viewport" content="width=device-width, initial-scale=1">
  <link rel="icon" type="image/x-icon" href="favicon.ico">
  <link href="https://fonts.googleapis.com/icon?family=Material+Icons" rel="stylesheet">
  <link href="https://fonts.googleapis.com/css?family=Roboto:300,400,500" rel="stylesheet">
  <link href="http://lib.baomitu.com/font-awesome/5.0.8/web-fonts-with-css/css/fontawesome-all.min.css" rel="stylesheet">
</head>

<body>
  <app-root></app-root>
</body>

</html>
```

#### 滑动开关

滑动开关需要导入 `MatSlideToggleModule` ，提供了一种可以通过点击或拖拽切换开/关状态的交互方式。

```html
<mat-slide-toggle (change)="toggleDarkMode($event.checked)" *ngIf="!hideForGuest">黑夜模式</mat-slide-toggle>
```

在 `HeaderComponent` 中，我们可以看到这个控件提供 `change` 事件，这个事件（输出型属性）的定义是 `@Output() change: EventEmitter<MatSlideToggleChange>` ，我们可以看到它的参数是 `MatSlideToggleChange`，有 2 个属性

1. checked -- 布尔型，代表最新的 `MatSlideToggle` 状态
2. source -- 产生事件的源控件

所以我们在事件处理中可以使用 `$event.checked` 将最新的状态传入处理函数。

### Footer （页尾）

页尾在这里就非常简单了，只是居中显示版权信息。

```ts
import { Component, ChangeDetectionStrategy } from '@angular/core';

@Component({
  selector: 'app-footer',
  template: `
  <mat-toolbar color="primary">
    <span class="fill-remaining-space"></span>
    <span>&copy; 2018 版权所有: 接灰的电子产品</span>
    <span class="fill-remaining-space"></span>
  </mat-toolbar>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class FooterComponent {}
```

### Sidebar （侧边栏）

我们在 `src/app/core/components` 中建立 `sidebar.component.ts` 。在侧边栏中，我们采用了另一个控件 `mat-nav-list` ，是列表控件的一种，这种控件一般用来展现一系列结构类似的数据。`Angular Material` 提供了这种控件的多种变化类型：

* 简单列表 -- `<mat-list>`  用于展现列表数据
* 导航列表 -- `<mat-nav-list>` 用于点击后导航到不同的视图
* 选择列表 -- `<mat-selection-list>` 用于选择多项数据

此外，还提供了一系列指令，可以配合列表类型实现更多效果

* `matLine` -- 用于实现多行的列表条目
* `matListIcon` -- 用于给条目添加图标
* `matListAvatar` -- 给列表条目增加头像支持
* `dense` -- 形成紧凑排布的列表

下面的两个指令不是专门用于 `List` 的：

* `mat-divider` -- 一条横线，用于分隔内容。我们在这个例子里面用于菜单分组。默认粗度 `1dp` ，依据主题的明暗，透明度分别是 `12% black` 或 `12% white` 。
* `matSubheader` -- 用于列表分组标题的显示，一般用于 `List`，`Grid` 和 `Menu` 中，默认 `48dp` 。

```ts
import {
  Component,
  ChangeDetectionStrategy,
  Input,
  Output,
  EventEmitter
} from '@angular/core';
import { MenuGroup, IconType } from '../../domain/menu';

@Component({
  selector: 'app-sidebar',
  template: `
  <mat-nav-list>
    <ng-container *ngFor="let group of menuGroups">
      <h3 matSubheader> {{ group.name }} </h3>
      <ng-container *ngFor="let item of group.items">
        <mat-list-item
          [routerLink]="item.routerLink"
          (click)="menuClick(item.emitData)" [ngSwitch]="item.iconType">
          <mat-icon matListIcon *ngSwitchCase="iconType.SVG" [svgIcon]="item.iconName"></mat-icon>
          <mat-icon matListIcon *ngSwitchCase="iconType.FONT_AWESOME" [fontSet]="item?.fontSet" [fontIcon]="item.iconName"></mat-icon>
          <mat-icon matListIcon *ngSwitchDefault>{{ item.iconName }}</mat-icon>
          <h4 matLine> {{ item.title }} </h4>
          <p matLine> {{ item.subtitle }} </p>
        </mat-list-item>
      </ng-container>
      <mat-divider></mat-divider>
    </ng-container>
  </mat-nav-list>
  `,
  styles: [
    `
  .day-num {
    font-size: 48px;
    width: 48px;
    height: 48px;
  }
  `
  ],
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class SidebarComponent {
  @Input() menuGroups: MenuGroup[] = [];
  @Output() menuClickEvent = new EventEmitter<any>();

  menuClick(data: any) {
    if (data) {
      this.menuClickEvent.emit(data);
    } else {
      this.menuClickEvent.emit();
    }
  }
  // 前面的 get 关键字表示这个方法是一个属性方法，可以当成属性使用
  get iconType() {
    return IconType;
  }
}
```

这个侧边栏组件的内容不是很多，但里面有一些值得注意的知识点

1. `get` 关键字。 `TypeScript` 提供 `get` 和 `set` 关键字用于标识属性方法，一般如果是方法的话，在模版中调用，需要写上括号和参数等，比如上面的 `iconType` 如果按照方法写的话应该写成 `iconType()` 。但属性方法可以直接当成属性使用，就可以直接写成 `iconType`。模版中无法直接访问 `IconType` 这个类型，这也是我们为什么用一个属性方法去得到它。
2. `ngSwitch` 的使用。枚举中有三个值，这时候使用 `ngSwitch` 这种分支条件指令就很方便。
3. 这又是一个“笨组件”，我们可以看到这个组件不关心你具体的业务，只是渲染菜单，这种形式的可扩展性和可维护性都非常好。我们只需要在使用 `<app-sidebar>` 的地方去构造一个 `MenuGroup` 数组，并赋值给组件的 `menuGroups` 属性即可。

由于菜单的数据类型比较相似，我们构造了 `MenuGroup` 和 `MenuItem` 来组织菜单的数据结构。这个文件位于 `src/app/domain` 中，我们创建一个 `menu.ts` 用来定义菜单相关的数据结构。一个菜单条目包含菜单的标题、副标题、图标名称和要导航的链接。而一个菜单分组会包含多个菜单项。而每个菜单项可以是 `Material Icon` 、 `FontAwesome` 或者是 `svg` 类型的图标，所以我们定义了一个枚举 `IconType` 用来标识类别。而特殊的对于 `FontAwesome` 的图标，有一个可选属性 `fontSet`，定义中的 `?` 表示这个属性设置成可选。

```ts
export enum IconType {
  MATERIAL_ICON,
  FONT_AWESOME,
  SVG
}

export interface MenuItem {
  routerLink: string[];
  iconName: string;
  iconType: IconType;
  fontSet?: string;
  title: string;
  subtitle: string;
  emitData: any;
}

export interface MenuGroup {
  name: string;
  items: MenuItem[];
}

```

#### 实验效果

我们要看看效果的话，就需要在 `AppComponent` 中使用这些组件，并为其设置必要的属性或处理产生的事件。首先在模版中，我们使用 `app-header` 、 `app-footer` 以及 `app-sidebar`

```html
<mat-sidenav-container fullscreen>
  <mat-sidenav #sidenav mode="over">
    <app-sidebar [menuGroups]="sidebarMenu"></app-sidebar>
  </mat-sidenav>
  <div class="site" fxLayout="column">
    <app-header (toggleMenuEvent)="sidenav.toggle()"></app-header>
    <main fxFlex="1" fxLayout="column" fxLayoutAlign="start center" fxLayoutGap="10px">
      <router-outlet></router-outlet>
    </main>
    <app-footer></app-footer>
  </div>
</mat-sidenav-container>
```

在 `app-header` 中我们需要处理图标按钮点击事件，这个事件在 `app-header` 中定义成了 `toggleMenuEvent` ，我们在这个事件发生时需要切换 `sidenav` 的显示和隐藏。我们是给 `mat-sidenav` 定义了一个引用名 `#sidenav`，这样后面就可以使用 `sidenav` 引用这个组件了。 `(toggleMenuEvent)="sidenav.toggle()"` 这句的意思就是当 `toggleMenuEvent` 发生时，我们就调用 `sidenav.toggle()` 切换侧边栏的显示和隐藏。

而在 `app-sidebar` 中，我们设置了 `[menuGroups]="sidebarMenu"` ，这个 `sidebarMenu` 是什么鬼？别急，我们还没有定义这个成员变量，接下来我们来看一下它的定义：

```ts
import { Component } from '@angular/core';
import { MenuGroup, IconType } from '../../../domain/menu';

@Component({
  selector: 'app-root',
  templateUrl: './app.component.html',
  styleUrls: ['./app.component.scss']
})
export class AppComponent {
  isCollapsed = true;
  sidebarMenu: MenuGroup[] = [];
  constructor() {
    this.sidebarMenu = [
      {
        name: '项目列表',
        items: [
          {
            title: '项目一',
            subtitle: '项目一的描述',
            iconName: 'project',
            iconType: IconType.SVG,
            emitData: 'abc123',
            routerLink: ['/projects/abc123']
          },
          {
            title: '项目二',
            subtitle: '项目二的描述',
            iconName: 'home',
            iconType: IconType.MATERIAL_ICON,
            emitData: 'abc234',
            routerLink: ['/projects/abc234']
          },
          {
            title: '项目三',
            subtitle: '项目三的描述',
            iconName: 'fa-bell',
            iconType: IconType.FONT_AWESOME,
            fontSet: 'fontawesome',
            emitData: 'abc345',
            routerLink: ['/projects/abc345']
          }
        ]
      }
    ];
  }
}
```

在看效果之前，我们还需要做几件事，第一，我们还没有做路由模块，也就是说现在我们并没有告诉 Angular 什么链接应该显示什么组件。所以我们还需要在 `core` 中建立一个根路由模块 `app-routing.module.ts`，这个根路由模块决定了我们顶层的路由设计，后面无论我们是采用预加载方式还是懒加载方式，一级的路由定义都会在这个模块中。

```ts
import { NgModule } from '@angular/core';
import { RouterModule, Routes } from '@angular/router';
import { PageNotFoundComponent } from './components/page-not-found.component';

const routes: Routes = [
  {
    path: '**', component: PageNotFoundComponent
  }
];

@NgModule({
  imports: [RouterModule.forRoot(routes)],
  exports: [RouterModule]
})
export class AppRoutingModule {
}
```

大家可能会注意到上面我们定义了一个路由，对所有链接都让一个 `PageNotFoundComponent` 进行处理。理论上我们应该对于任何不存在的链接，让这个页面出现，但暂时的，我们让所有链接都指向它。在 `src/app/core/components` 中新建一个 `page-not-found.component.ts`

```ts
import { Component, OnInit, ChangeDetectionStrategy } from '@angular/core';

@Component({
  template: `
  <div fxLayout="row" fxLayoutAlign="center center">
    <mat-card>
      <mat-card-header>
        <mat-card-title> 又迷路了... </mat-card-title>
        <mat-card-subtitle> 404 - 没有找到页面 </mat-card-subtitle>
      </mat-card-header>
      <img mat-card-image [src]="notFoundImgSrc">
      <mat-card-actions>
        <button mat-button color="primary"> 回首页 </button>
      </mat-card-actions>
    </mat-card>
  </div>
  `,
  styles: [
    `
    :host {
      display: flex;
      flex: 1 1 auto;
    }
    mat-card {
      width: 70%;
    }
    `
  ],
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class PageNotFoundComponent {
  notFoundImgSrc = 'assets/img/400_night_light.jpg';
}
```

这个页面非常简单就是建立了一个带图片的卡片，在卡片的标题和副标题中写上让访问者理解目前处境的话。在 Material Design 中，卡片是显示内容的非常基本的一种形式。

组件 | 功能描述
---------|----------
`<mat-card-title>`|卡片标题
`<mat-card-subtitle>`|卡片副标题
`<mat-card-content>`|卡片内容，最好是文本内容
`<img mat-card-image>`|卡片图片，默认拉伸图片到容器宽度
`<mat-card-actions>`|卡片下方的按钮容器
`<mat-card-footer>`|卡片的尾部
`<mat-card-header>`|对于更复杂的卡片头部，可以采用这个组件定制化，里面可以含有 `<mat-card-title>` 、 `<mat-card-subtitle>` 和 `<img mat-card-avatar>`
`<mat-card-title-group>`|可以把标题（`<mat-card-title>`）、副标题（`<mat-card-subtitle>`）和图片（`<img mat-card-sm-image>` 或 `<img mat-card-md-image>` 或 `<img mat-card-lg-image>`）合并进一个区块。


使用卡片，我们需要导入 `MatCardModule`，所以我们要更改 `src/app/core/material.module.ts` ，加上这个模块。这样一个个的添加很麻烦，我们这里就偷点懒，索性一次性的把所有的 `Material` 组件都加上，但请在正式项目中只包含你实际用到的模块。

```ts
import { NgModule } from '@angular/core';
import {
  MatAutocompleteModule,
  MatButtonModule,
  MatButtonToggleModule,
  MatCardModule,
  MatCheckboxModule,
  MatDatepickerModule,
  MatDialogModule,
  MatExpansionModule,
  MatFormFieldModule,
  MatGridListModule,
  MatChipsModule,
  MatIconModule,
  MatInputModule,
  MatListModule,
  MatMenuModule,
  MatPaginatorModule,
  MatProgressBarModule,
  MatProgressSpinnerModule,
  MatRadioModule,
  MatSelectModule,
  MatSidenavModule,
  MatSliderModule,
  MatSlideToggleModule,
  MatSnackBarModule,
  MatSortModule,
  MatTableModule,
  MatTabsModule,
  MatToolbarModule,
  MatTooltipModule,
  MatStepperModule,
  MatTreeModule,
  MatBadgeModule,
  MatBottomSheetModule
} from '@angular/material';
import { MatNativeDateModule, MatRippleModule } from '@angular/material';
import { CdkTableModule } from '@angular/cdk/table';
import { CdkAccordionModule } from '@angular/cdk/accordion';
import { A11yModule } from '@angular/cdk/a11y';
import { BidiModule } from '@angular/cdk/bidi';
import { OverlayModule } from '@angular/cdk/overlay';
import { PlatformModule } from '@angular/cdk/platform';
import { ObserversModule } from '@angular/cdk/observers';
import { PortalModule } from '@angular/cdk/portal';
import { MatMomentDateModule } from '@angular/material-moment-adapter';
import { FlexLayoutModule } from '@angular/flex-layout';

/**
 * NgModule that includes all Material modules.
 */
@NgModule({
  exports: [
    MatAutocompleteModule,
    MatButtonModule,
    MatButtonToggleModule,
    MatCardModule,
    MatCheckboxModule,
    MatChipsModule,
    MatTableModule,
    MatDatepickerModule,
    MatDialogModule,
    MatExpansionModule,
    MatFormFieldModule,
    MatGridListModule,
    MatIconModule,
    MatInputModule,
    MatListModule,
    MatMenuModule,
    MatPaginatorModule,
    MatProgressBarModule,
    MatProgressSpinnerModule,
    MatRadioModule,
    MatRippleModule,
    MatSelectModule,
    MatSidenavModule,
    MatSlideToggleModule,
    MatSliderModule,
    MatSnackBarModule,
    MatSortModule,
    MatStepperModule,
    MatTabsModule,
    MatToolbarModule,
    MatTooltipModule,
    MatNativeDateModule,
    CdkTableModule,
    A11yModule,
    BidiModule,
    CdkAccordionModule,
    ObserversModule,
    OverlayModule,
    PlatformModule,
    PortalModule,
    MatTreeModule,
    MatBadgeModule,
    MatBottomSheetModule,
    MatMomentDateModule,
    FlexLayoutModule
  ]
})
export class MyMaterialModule {}
```

接下来就可以启动我们的应用了。在应用根目录下 `ng serve` 或者 `yarn start` 或者 `npm start` ，你应该可以看到下面的页面了。

![第一个页面](/assets/2018-04-07-14-28-21.png)
