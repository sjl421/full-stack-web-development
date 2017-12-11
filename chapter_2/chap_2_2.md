# Angular Material 介绍

Angular Material 是 Angular 团队官方开发的一套符合 Google Material 风格的 Angular UI 组件库。这套组件的使用方式上常常让我联想起 Android 的开发，个人感觉这应该也是 Google 努力的方向之一吧 -- 让 web 开发更像 app 开发或者后端开发。

`@angular/material` 在 `2.0.0-beta.8` 之前是单独的一个 `package`，但后来团队把其中的一些公用功能以及组件抽离出来放到了一个单独的 `@angular/cdk` 包中。这个 `cdk` 以后可以作为你开发自己风格组件库的基础，因为它封装了很多公共特性的支持，你不需要从零开始。

总体来说，`@angular/material` 提供了 30 多个组件以及主题和字体的支持，并通过 `@angular/flex-layout` 提供了 `flexbox` 布局的 angular 封装。

* 组件
  * 表单控件类：`Autocomplete`
  * 导航类：
  * 布局类：
  * 按钮和提示类
  * 弹出类：
  * 数据表格类：
* 布局支持：通过独立的 `@angular/flex-layout` 提供，这个软件包不仅提供 flex 的封装，也提供响应式页面设计需要的各种 API 和指令。
* 主题支持：主题的支持主要由框架提供的一系列 `scss` 函数来实现，因此如果希望有主题的自定义时，需要以 `scss` 形式提供样式。

##
