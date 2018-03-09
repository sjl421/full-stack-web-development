# 添加主题支持

Angular Material 是一个从诞生起就有主题支持的类库，既然叫 `Material` ，它的主题也是要受 `Material Design` 设计风格约束的。

## Material Design 中对于主题的约束

`Material Design` 中的颜色主要调色板（palette）来提供，值得注意的是， `Material Design` 并不鼓励使用太多颜色，而相反的提倡对颜色进行限制。调色板以一些基础色为基准，通过填充光谱来为Android、Web 和 iOS 环境提供一套完整可用的颜色，基础色的饱和度是 500。

构成一个主题的颜色需要在众多基础色中选出同一颜色系的三个色度以及一个强调色（可区别与前一种颜色），强调色用于某些元素的强调。

Google 的 `Material Design` 团队提供了一个在线调色工具，大家可以自行前往去体会一下 https://material.io/color/#!/?view.left=0&view.right=0

![Color Tool](/assets/2018-03-02-23-45-12.png)

如果我们严格的执行这种约束，那么即使你自定义了颜色，也会一样看上去就是 `Material Design` 风格。

## 主题的明与暗

`Material Design` 认为主题是对应用提供一致性色调的方法。样式指定了表面的亮度、阴影的层次和字体元素的适当不透明度。为了提高应用间的一致性，提供两种明暗的选择：`浅色` 和 `深色` 两种 (即 `light` 和 `dark` )。

换句话说即使选择了同样的基础色和强调色，你的主题也会有偏亮和偏暗两种选择。比如你想实现白天模式和黑夜模式的话，那么这两种模式就有明暗的区别。

## Angular Material 中的主题

知道了以上基础概念后，就比较好理解 Angular Material 中的主题了。 Angular Material 的主题由以下几部分构成：

* 基础色调色板：用于大部分的组件
* 强调色调色板：用于浮动按钮和交互元素
* 警告色调色板：用于显示错误状态，一般为红色
* 前景色调色板：用于文字和图标的颜色
* 背景色调色板：用于元素背景的颜色

在 Angular Material 中，主题在编译时被静态的输出生成。

### 内建主题

Angular Material 提供了几个官方配置好的主题：

* deeppurple-amber.css
* indigo-pink.css
* pink-bluegrey.css
* purple-green.css

以上内建主题都在 `@angular/material/prebuilt-themes` 目录下，所以一般来说，在你的 `styles.css` 或者 `styles.scss` 中导入对应主题，就可以将主题应用于你的所有 Material 组件了。

```css
@import '~@angular/material/prebuilt-themes/deeppurple-amber.css';
```

### 自定义主题

在很多情况下，内建主题是不能满足需求的，那么 `Angular Material` 也是允许你自定义主题的。

我们为主题风格单独建立一个文件 -- theme.scss，在 style.scss 中导入 theme.scss

```css
// styles.scss
@import 'theme.scss';
```

下面我们来看看如何写一个主题

```scss
// theme.scss
@import '~@angular/material/theming';

// mat-core 提供了 Angular Material 的公共风格
// 这个 mat-core 只能在整个应用中包含一次
@include mat-core();

// 使用 mat-palette 添加基础色和强调色调色板
// 对于每种调色板，你可以手动指定三种饱和度，分别对应 默认、亮和暗
// $mat-indigo 和 $mat-pink 是 Material 定义的基本色中的两种，具体都有哪些颜色可以查看
// node_modules/@angular/material/_theming.scss
// 如果还是不满足你的需求你可以自己定义颜色变量
/**
$mat-custom: (
  50: #fce4ec,
  100: #f8bbd0,
  200: #f48fb1,
  300: #f06292,
  400: #ec407a,
  500: #e91e63,
  600: #d81b60,
  700: #c2185b,
  800: #ad1457,
  900: #880e4f,
  A100: #ff80ab,
  A200: #ff4081,
  A400: #f50057,
  A700: #c51162,
  contrast: (
    50: $dark-primary-text,
    100: $dark-primary-text,
    200: $dark-primary-text,
    300: $dark-primary-text,
    400: $dark-primary-text,
    500: $light-primary-text,
    600: $light-primary-text,
    700: $light-primary-text,
    800: $light-primary-text,
    900: $light-primary-text,
    A100: $dark-primary-text,
    A200: $light-primary-text,
    A400: $light-primary-text,
    A700: $light-primary-text,
  )
);
**/
$app-primary: mat-palette($mat-indigo);
$app-accent:  mat-palette($mat-pink, A200, A100, A400);

// 错误调色板，默认为红色
$app-warn:    mat-palette($mat-red);

// 通过 mat-light-theme 将上面定制化的调色板组合成一个主题，注意主题分为明和暗两种
// mat-light-theme：明
// mat-dark-theme：暗
$app-theme: mat-light-theme($app-primary, $app-accent, $app-warn);

// 接下来这句就是把主题风格包含以便组件可以使用
@include angular-material-theme($app-theme);
```

必须注意的是，一般情况下，主题的样式只需导入一次，不要在多个文件中导入，以免样式风格被重复生成。

### 如何包含多个主题

有的时候，我们可能希望有多个主题，方便用户切换，或者不同模块使用不同主题。这种情况下，可以使用多个 `css` 类多次包含 `angular-material-theme` 来处理。

```scss
.app-dark-theme {
  @include angular-material-theme($dark-theme);
}

.app-other-theme {
  @include angular-material-theme($other-theme);
}
```

在对应组件的样式中应用该样式即可

```scss
mat-sidenav-container.myapp-dark-theme {
  background: black;
}
```

### 对于特殊组件的支持

在 Angular Material 中有一些组件是比较特殊的，比如对话框、弹出式菜单等，这一类组件是动态创建的，而且一般是要覆盖在其他组件之上显示的，这一类组件在 Angular Material 中都是 `OverlayContainer` 。对于这类组件，要支持主题的话，需要多做一点工作

* 在构造函数中注入 `OverlayContainer`
* 然后在对应的切换主题或者应用主题的函数中，使用上面得到的对象的 `getContainerElement().classList.add` 和 `getContainerElement().classList.remove` 方法来添加或删除主题的支持。

```ts
constructor(private oc: OverlayContainer) {}

// 切换主题
switchDarkTheme(dark: boolean) {
    if (dark) {
      // 将定义的主题类应用到 OverlayContainer
      this.oc.getContainerElement().classList.add('myapp-dark-theme');
    } else {
      // 将定义的主题类从 OverlayContainer 的风格列表中移除
      this.oc.getContainerElement().classList.remove('myapp-dark-theme');
    }
  }
```
