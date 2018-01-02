# Angular Material 介绍

Angular Material 是 Angular 团队官方开发的一套符合 Google Material 风格的 Angular UI 组件库。这套组件的使用方式上常常让我联想起 Android 的开发，个人感觉这应该也是 Google 努力的方向之一吧 -- 让 web 开发更像 app 开发或者后端开发。

`@angular/material` 在 `2.0.0-beta.8` 之前是单独的一个 `package`，但后来团队把其中的一些公用功能以及组件抽离出来放到了一个单独的 `@angular/cdk` 包中。这个 `cdk` 以后可以作为你开发自己风格组件库的基础，因为它封装了很多公共特性的支持，你不需要从零开始。

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

