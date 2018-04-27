# 模块化和组件化

不论是企业级的应用还是面向消费市场的应用，一般都会有登录、注册等功能。我们要构建的是一个团队效率工具，所以这个功能也是不可缺少的。

## 登录的领域模型构建

开始之前需要注意，我们会把领域对象都建立在 `src/app/domain` 这个文件夹下面，一般情况下每个领域对象建一个文件，当然有时几个对象关系特别紧密时，我们也会把它们统一放在一个文件中。一个比较基础的用户对象是下面的样子

```ts
export interface User {
  // 唯一标识
  id: string;
  // 用户名
  login: string;
  // 姓名
  name: string;
  // 密码
  password?: string
  // 手机号
  mobile: string;
  // 头像
  avatar: string;
  // 电子邮件
  email: string | null;
}
```

有的同学可能会注意到，密码这个字段是可选的。这个是由于密码属于比较敏感的字段，一般在商用系统中不会暴露出来，即便在服务器端已经加密存储了。所以从服务器读取用户信息时一般不会返回此字段。

这样一个对象在注册时或者登录成功后返回时是比较适合的，但对于登录本身，我们其实只需用户名和密码即可，那么我们就再新建一个 `Auth` 对象。为什么不复用 `User` 呢，这里就涉及到编程中的一个重要原则：单一责任原则。任何对象或者方法都应该只负责一件事。鉴权这件事其实不是 `User` 对象应该负责的，因为鉴权时我还不知道你是否是我的用户，我需要判断你的用户名和密码之后才能确定你是系统用户。而这个判断中我们只不过恰好需要 `User` 中的用户名和密码两个字段而已。

```ts
export interface Auth {
  login: string;
  password: string;
}
```

那么有同学又有疑问了，为什么 `Auth` 搞了个 `login` 的属性，而不是直接写 `username` 呢？这个呢，是由于我们之后可能会有不止一种的登录方式，比如用户可不可以使用手机号登录呢？手机号登录现在基本是国内登录的标配了。那么 email 登陆或社交登录呢？对的，为了以后不用频繁改名字，我们就叫这个登录名 `login` 好了。

除了这两个对象还有什么对象没有？有的，但是这个不是登录必须的，而是我自己添加的。因为干巴巴的一个登录，显得好无趣啊，页面也很空，所以我想作为一个效率工具，给每个人每天加加油，给一段名人名言不也挺好吗？每次登录的时候给自己打点鸡血，哈哈！

于是就有了这个“佳句” `Quote` 对象：我们会有一个图片的链接、一个中文的佳句（或者英文的翻译）和一个英文的佳句（或对中文的翻译）。

```ts
export interface Quote {
  cn: string;
  en: string;
  imgUrl: string;
}
```

## 前端页面设计

在开始构建前端之前，为了让各位有个初步的印象，我们先看看成品页面。

登录页面由两个卡片构成，左边是佳句，右边的是登录表单。登录表单下部有两个路由链接，可以跳转到注册和忘记密码页面。

![登录页面](/assets/2018-04-09-00-25-45.png)

注册页面比登录页面要复杂，复杂主要体现在表单验证比较复杂，包括必填字段验证、最小长度验证、最大长度验证、正则表达式验证以及异步验证用户名、手机和电子邮件是否唯一等。这个页面还涉及到一个自定义表单控件，就是选取头像的这个控件。

![注册页面](/assets/2018-04-09-00-26-52.png)

忘记密码页面我们采用了 Angular Material 的 Stepper 组件开发，Stepper 类似一个向导页面，一步步的指引用户完成某个特定的业务。这个组件里面我们会使用 2 个表单，也就是每个步骤都是一个表单。而且里面还有一个较复杂的自定义控件，就是发送短信验证码的这个控件。

![忘记密码页面](/assets/2018-04-09-07-49-12.png)

### 使用 CLI 生成模块

在 `frontend` 目录下键入

```bash
ng g m auth --routing
```

`Angular CLI` 就会为我们生成 3 个文件，下面的就是执行命令后的输出，你会发现多了一个 `auth` 目录，这个目录里面就是这三个文件。

```log
CREATE src/app/auth/auth-routing.module.ts (247 bytes)
CREATE src/app/auth/auth.module.spec.ts (259 bytes)
CREATE src/app/auth/auth.module.ts (271 bytes)
```

其中 `auth.module.ts` 是 Auth 模块文件，`auth.module.spec.ts` 是模块测试文件，而 `auth-routing.module.ts` 是这个路由子模块文件。对于 `auth.module.ts` 我们把默认生成的 `imports` 数组中的 `CommonModule` 去掉，替换成 `SharedModule` 。

```ts
import { NgModule } from '@angular/core';
import { AuthRoutingModule } from './auth-routing.module';
import { SharedModule } from '../shared/shared.module';

@NgModule({
  imports: [SharedModule, AuthRoutingModule]
})
export class AuthModule {}
```

接下来，我们在 `auth` 目录下创建 3 个子目录

* `components` 目录 -- “笨组件”存放在这个目录，这个目录中组件，请记得设置 `changeDetection: ChangeDetectionStrategy.OnPush`
* `containers` 目录 -- “聪明组件”存放在这个目录
* `services` 目录 -- 服务和路由守卫存放在此目录

### 创建“笨组件”

我们本章涉及的笨组件都位于 `src/app/auth/components` 目录中，如果没有这个目录，请手动创建它。这也会是我们后期前端工程中的一个规范 -- 任何一个模块中的 `components` 目录中都是这个模块的笨组件。

#### 登录表单组件

首先我们还是创建一个 `login-form` 的笨组件，注意下面命令中的 `--module auth/auth.module.ts` 选项可以在 `AuthModule` 中声明新创建的组件。

```bash
ng g c auth/components/login-form --module auth/auth.module.ts
```

这个命令会在 `auth/components` 目录下新创建一个叫 `login-form` 的目录，此目录下会生成 4 个文件

* `login-form.component.ts` -- 组件文件
* `login-form.component.html` -- 组件模版文件
* `login-form.component.scss` -- 组件样式文件
* `login-form.component.spec.ts` -- 组件单元测试文件

那么我们首先更改其模版，模版是一个卡片布局：卡片的头部有标题和副标题；内容部分是一个简单的响应式表单，有 `登录名` 和 `密码` 两个输入框；而卡片的尾部操作部分是两个链接，可以导航到注册页面或者忘记密码页面。

```html
<mat-card>
  <mat-card-header>
    <mat-card-title> {{ title }} </mat-card-title>
    <mat-card-subtitle> {{ subtitle }} </mat-card-subtitle>
  </mat-card-header>
  <mat-card-content>
    <form fxLayout="column" fxLayoutAlign="stretch center" [formGroup]="form" (ngSubmit)="submit(form, $event)">
      <mat-form-field class="full-width">
        <input matInput type="text" placeholder="用户名" formControlName="login">
        <mat-error>用户名是必填项哦</mat-error>
      </mat-form-field>
      <mat-form-field class="full-width">
        <input matInput type="password" placeholder="您的密码" formControlName="password">
        <mat-error>密码不正确哦</mat-error>
      </mat-form-field>
      <div fxLayout="row" fxLayoutAlign="end stretch" class="full-width">
        <button mat-raised-button color="primary" type="submit" [disabled]="!form.valid">登录</button>
      </div>
    </form>
  </mat-card-content>
  <mat-card-actions>
    <div fxLayout="row" fxLayoutAlign="end stretch">
      <a mat-button routerLink="/auth/register"> {{ regBtnText }} </a>
      <a mat-button routerLink="/auth/forgot"> {{ forgotBtnText }} </a>
    </div>
  </mat-card-actions>
</mat-card>
```

对应的组件文件中定义了页面显示的几个文本的输入型属性，以及一个事件 `submitEvent` ，用于在表单提交时将表单数据发送给调用者。

```ts
import {
  Component,
  OnInit,
  ChangeDetectionStrategy,
  EventEmitter,
  Output,
  Input
} from '@angular/core';
import { FormGroup, FormBuilder, Validators } from '@angular/forms';
import { usernamePattern } from '../../../utils/regex';
import { Auth } from '../../../domain/auth';

@Component({
  selector: 'app-login-form',
  templateUrl: './login-form.component.html',
  styleUrls: ['./login-form.component.scss'],
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class LoginFormComponent implements OnInit {
  form: FormGroup;
  @Input() title = '登录';
  @Input() subtitle = '使用您的用户名密码登录';
  @Input() regBtnText = '还没有注册？';
  @Input() forgotBtnText = '忘记密码？';
  @Output() submitEvent = new EventEmitter<Auth>();

  constructor(private fb: FormBuilder) {}

  ngOnInit() {
    this.form = this.fb.group({
      login: [
        '',
        Validators.compose([
          Validators.required,
          Validators.pattern(usernamePattern)
        ])
      ],
      password: [
        '',
        Validators.compose([
          Validators.required,
          Validators.minLength(8),
          Validators.maxLength(50)
        ])
      ]
    });
  }

  submit({ value, valid }: FormGroup, ev: Event) {
    if (!valid) {
      return;
    }
    this.submitEvent.emit(value);
  }
}
```

#### 佳句组件

我们提到过还需要有一个佳句，在登录页面，我们希望这个佳句也是以卡片形式提供，和登录的卡片并列放置。类似的，我们使用 CLI 创建一个 `Quote` 组件：

```bash
➜ ng g c auth/components/quote --module auth/auth.module.ts
CREATE src/app/auth/components/quote/quote.component.scss (0 bytes)
CREATE src/app/auth/components/quote/quote.component.html (24 bytes)
CREATE src/app/auth/components/quote/quote.component.spec.ts (621 bytes)
CREATE src/app/auth/components/quote/quote.component.ts (261 bytes)
UPDATE src/app/auth/auth.module.ts (1012 bytes)
```

更改 `Quote` 模版为一个卡片形式：中文为卡片头部的副标题，英文为卡片内容，而图片使用 `matCardImage` 指令拉伸到卡片宽度。

```html
<mat-card>
  <mat-card-header>
    <mat-card-title> {{ title }} </mat-card-title>
    <mat-card-subtitle> {{ chineseQuote}} </mat-card-subtitle>
  </mat-card-header>
  <img matCardImage [src]="quoteImg">
  <mat-card-content>
    <p> {{ englishQuote }}</p>
  </mat-card-content>
</mat-card>
```

而它的对应组件文件比较简单:

```ts
import { Component, Input } from '@angular/core';

@Component({
  selector: 'app-quote',
  templateUrl: './quote.component.html',
  styleUrls: ['./quote.component.scss']
})
export class QuoteComponent {
  @Input() title = '佳句';
  @Input() chineseQuote = '';
  @Input() englishQuote = '';
  @Input() quoteImg = '';
}
```

#### 注册表单组件

注册表单属于相对来说比较复杂的一个，下面的文件就是这个组件的模版。

```html
<mat-card>
  <mat-card-header>
    <mat-card-title> {{ title }} </mat-card-title>
    <mat-card-subtitle> {{ subtitle }} </mat-card-subtitle>
  </mat-card-header>
  <mat-card-content>
    <form [formGroup]="form" (ngSubmit)="submit(form, $event)">
      <mat-form-field class="full-width">
        <input matInput placeholder="您的用户名" formControlName="login">
        <mat-error> {{ usernameErrors }} </mat-error>
      </mat-form-field>
      <mat-form-field class="full-width">
        <input matInput placeholder="您的手机号" formControlName="mobile">
        <mat-error> {{ mobileErrors }}</mat-error>
      </mat-form-field>
      <mat-form-field class="full-width">
        <input matInput placeholder="电子邮件" formControlName="email">
        <mat-error> {{ emailErrors }}</mat-error>
      </mat-form-field>
      <mat-form-field class="full-width">
        <input matInput type="text" placeholder="您的名字" formControlName="name">
        <mat-error> {{ nameErrors }}</mat-error>
      </mat-form-field>
      <ng-container formGroupName="passwords">
        <mat-form-field class="full-width">
          <input matInput type="password" placeholder="您的密码" formControlName="password">
          <mat-error> {{ passwordErrors }} </mat-error>
        </mat-form-field>
        <mat-form-field class="full-width">
          <input matInput type="password" placeholder="为避免失误请再次输入" formControlName="repeat">
          <mat-error> {{ repeatErrors }}</mat-error>
        </mat-form-field>
      </ng-container>
      <div fxLayout="row" fxLayoutAlign="center stretch" class="full-width">
        <button mat-raised-button color="primary" type="submit" [disabled]="!form.valid">注册</button>
        <button mat-raised-button color="primary" type="reset">重新填写</button>
      </div>
    </form>
  </mat-card-content>
  <mat-card-actions>
    <div fxLayout="row" fxLayoutAlign="end stretch">
      <a mat-button routerLink="/auth/login"> {{ loginBtnText }} </a>
      <a mat-button routerLink="/auth/forgot"> {{ forgotBtnText }} </a>
    </div>
  </mat-card-actions>
</mat-card>
```

如果你仔细看一下的话，会发现我们漏掉了一个选择头像的功能。这是由于 `Angular Material` 内建组并没有提供，如果要实现的话，需要使用 `GridList` 和一些 `html` 标签以及对应逻辑完成。这样的一坨代码如果直接写在注册表单中的话，显然违背了“单一责任”原则。而且从代码的可读性来说也是非常不好的。

如果我们实现一个普通组件是否可以呢？当然会比前一种方案好很多，但是我们观察到表单控件比如 `input` 是可以通过指定其 `formControlName` ，然后在组建文件中初始化这个控件的，这个控件的验证状态和值变化等也可以通过 `control.stateChanges` 或 `control.valueChanges` 得到。那么我们有没有可能做成一个具备同样特性的控件呢？当然是可以了，我们就一起学习如果做一个表单控件。

我们首先用 CLI 创建一个组件，但这个组件放在 `auth` 模块中显然是有点问题的。因为这个组件的复用可能性比较高，所以应该放在 `shared` 模块中，这样所有模块都可以使用这个组件了。当然这个还是个笨组件，如果 `shared` 目录下还没有 `components` 子目录的话，请先创建。我们命名这个组件为 `ImagePicker` 。

```bash
ng g c shared/components/image-picker
```

注意这次我们没有指定 `--module` ，这是因为 `SharedModule` 中由于导入导出的东西太多，我们做了一些特殊处理，我们不希望 CLI 自动声明组件。所以我们只好手动添加一下。在 `shared.module.ts` 中更新 `COMPONENTS` 数组，添加 `ImagePicker` 。

```ts
// 省略其他部分
const COMPONENTS = [ImagePicker];

@NgModule({
  declarations: COMPONENTS,
  imports: [...MODULES],
  exports: [...MODULES, ...COMPONENTS]
})
export class SharedModule {}
```

然后先看一下我们要实现的效果，这个表单控件分为上下两部分，下面是一个备选图片区域，上面是一个选中图片的显示区域。鼠标划过备选图片区域的图片时，图片会叠加一个遮罩，用来提示用户鼠标指向当前的图片。而由于备选图片有时可能会很多，这样的话，这个备选区域应该是一个可滚动的区域。

![图片选择器的外观](/assets/2018-04-09-10-39-38.png)

知道了需求，我们看看设计一下组件的模版，这里我们使用了一个新的 `Angular Material` 组件 -- `MatGridList` 。这个控件和我们在 `html` 中见过的 `table` 有点像，都是以行列形式展现数据。而且数据可以通过指定 `colspan` 和 `rowspan` 来占据多列或多行。

![一个样例 GridList](/assets/2018-04-09-11-03-33.png)

上面的效果就是下面的模版产生的

```html
<mat-grid-list cols="3" rowHeight="2:1">
  <mat-grid-tile>1</mat-grid-tile>
  <mat-grid-tile colspan="2">2</mat-grid-tile>
  <mat-grid-tile rowspan="2">3</mat-grid-tile>
  <mat-grid-tile rowspan="2" colspan="2">4</mat-grid-tile>
</mat-grid-list>
```

我们可以看到有两个组件构成了 `GridList` ： `<mat-grid-list>` 和 `mat-grid-tile` 。 `<mat-grid-list>` 必须指定 `cols` 属性用来确定网格中的列数。

而每行的高度可以通过 `rowHeight` 属性进行设置，行高有以下几种形式

* 固定高度：可以是 `px` 、`em` 或者 `rem` 几种单位。
* 比例：这里的比例指的是 `列宽:行高`
* 自适应：设置为 `fit` 会依据 GridList 的高度或其容器高度自动为分配行数。

此外 `<mat-grid-list>` 还有一个 `gutterSize` 属性用来设置单元格直接的间隔。

知道这些我们就可以开始我们的组件模版设计了。

```html
<div>
  <span>{{ title }}</span>
  <mat-icon class="avatar" [svgIcon]="selected" *ngIf="useSvgIcon else imgSelected"></mat-icon>
  <ng-template #imgSelected>
    <img class="cover" [src]="selected">
  </ng-template>
</div>
<div class="scroll-container">
  <mat-grid-list [cols]="cols" [rowHeight]="rowHeight">
    <mat-grid-tile *ngFor="let item of items; let i = index;">
      <div class="image-container" (click)="selectImage(i)">
        <mat-icon class="avatar" [svgIcon]="item" *ngIf="useSvgIcon else imgItem"></mat-icon>
        <ng-template #imgItem>
          <img [src]="item" [ngStyle]="{'width': itemWidth }">
        </ng-template>
        <div class="after">
          <span class="zoom">
            <mat-icon>checked</mat-icon>
          </span>
        </div>
      </div>
    </mat-grid-tile>
  </mat-grid-list>
</div>
```

值得注意的一点是我们兼容了图标的选取而不是仅仅图片，这是由于两者的选取逻辑完全一致，仅仅是文件格式和使用的渲染控件上有所区别。

```html
<mat-icon class="avatar" [svgIcon]="selected" *ngIf="useSvgIcon else imgSelected">
</mat-icon>
<ng-template #imgSelected>
  <img class="cover" [src]="selected">
</ng-template>
```

上面这段代码在 `Angular` 中是非常常见的范式 -- 如果怎样就渲染什么，否则就渲染什么。而在 `*ngIf="useSvgIcon else imgSelected"` 中使用的 `else` 指定的是一个模版引用名 `imgSelected` ，就是下面 `ng-template` 的引用名。

敲黑板，重点来了，我们看看一个表单控件应该怎么写。Angular 中提供了一个 `ControlValueAccessor` 接口，实现这个接口并注册为 `NG_VALUE_ACCESSOR` 就可以做出一个表单控件了。

在具体介绍这个接口之前，我们先回顾一些 Angular 中的 `FormControl` 的概念。 `FormControl` 是一个用于追踪表单控件的值和验证状态的实体类，无论你使用模版驱动型表单还是响应式表单，这个 `FormControl` 都会被创建。只不过在响应式表单中你要指定 `formControlName` 指令去绑定这个 `FormControl` 到一个 HTML 原生控件；而在模版驱动型表单中，这个绑定过程是隐含的，通过 `ngModel` 的绑定。因为 `ngModel` 的这个指令内部初始化了 `FormControl` ，下面的 `ngModel` 源码片段告诉了我们原因。

```ts
@Directive({
  selector: '[ngModel]...',
  ...
})
export class NgModel ... {
  _control = new FormControl();   <---------------- 这里
```

所以呢， `FormControl` 是 Angular 中追踪和改变 HTML 原生控件的方式。而 `ControlValueAccessor` 接口就是 Angular Form API 和 HHTML 原生控件直接互相交互以及状态同步的桥梁。

那么接下来我们回到要实现的表单控件，如果现在只是实现一个普通控件的话，其组件代码如下，非常简单，不做解释了。

```ts
import {
  Component,
  Input,
  Output,
  EventEmitter
} from '@angular/core';

@Component({
  selector: 'app-image-picker',
  templateUrl: './image-picker.component.html',
  styleUrls: ['./image-picker.component.scss']
})
export class ImagePicker {
  selected: string | null = null;
  @Input() title = '选择封面：';
  @Input() items: string[] = [];
  @Input() cols = 8;
  @Input() rowHeight = '64px';
  @Input() itemWidth = '80px';
  @Input() useSvgIcon = false;
  @Output('itemChange') itemChange = new EventEmitter<string>();
  // 列表元素选择发生改变触发
  selectImage(i: number) {
    this.selected = this.items[i];
    this.itemChange.emit(this.items[i]);
  }
}
```

那么我们现在要做的就是将这个普通控件转化成表单控件，改造后的代码如下：

```ts
import {
  Component,
  Input,
  Output,
  EventEmitter,
  forwardRef
} from '@angular/core';
import {
  ControlValueAccessor,
  NG_VALIDATORS,
  NG_VALUE_ACCESSOR,
  FormControl
} from '@angular/forms';

@Component({
  selector: 'app-image-picker',
  templateUrl: './image-picker.component.html',
  styleUrls: ['./image-picker.component.scss'],
  providers: [
    {
      provide: NG_VALUE_ACCESSOR,
      useExisting: forwardRef(() => ImagePicker),
      multi: true
    },
    {
      provide: NG_VALIDATORS,
      useExisting: forwardRef(() => ImagePicker),
      multi: true
    }
  ]
})
export class ImagePicker implements ControlValueAccessor {
  selected: string | null = null;
  @Input() title = '选择封面：';
  @Input() items: string[] = [];
  @Input() cols = 8;
  @Input() rowHeight = '64px';
  @Input() itemWidth = '80px';
  @Input() useSvgIcon = false;
  @Output('itemChange') itemChange = new EventEmitter<string>();

  // 这里是做一个空函数体，真正使用的方法在 registerOnChange 中
  // 由框架注册，然后我们使用它把变化发回表单
  // 注意，和 EventEmitter 尽管很像，但发送回的对象不同
  private propagateChange = (_: any) => {};

  // 写入控件值
  public writeValue(obj: any) {
    this.selected = obj ? String(obj) : null;
  }

  // 当表单控件值改变时，函数 fn 会被调用
  // 这也是我们把变化 emit 回表单的机制
  public registerOnChange(fn: any) {
    this.propagateChange = fn;
  }

  // 验证表单，验证结果正确返回 null 否则返回一个验证结果对象
  public validate(c: FormControl) {
    return this.selected
      ? null
      : {
          imageListSelect: {
            valid: false
          }
        };
  }

  // 这里没有使用，用于注册 touched 状态
  public registerOnTouched() {}

  // 列表元素选择发生改变触发
  selectImage(i: number) {
    this.selected = this.items[i];
    // 更新表单
    this.propagateChange(this.items[i]);
    this.itemChange.emit(this.items[i]);
  }
}
```

实现 `ControlValueAccessor` 的话，需要实现 4 个方法

* `registerOnChange(fn)` -- 这个函数中传入的是值变化的回调函数 `fn` ，我们需要在本地保存它的引用 ( `this.propagateChange = fn` )，并在控件值变化的时候调用这个本地引用，将值传递出去 `this.propagateChange(this.items[i])`
* `writeValue(obj)` -- 外部写入控件值的时候会调用此方法，所以我们把外部传入的值写入控件 `this.selected = obj ? String(obj) : null;`
* `registerOnTouched(fn)` -- 传入是否点击控件的回调函数，如果需要的话，也像 `registerOnChange(fn)` 的处理那样做一个本地保存。这个函数的目的是设置一个当控件接受到点击事件时可以被调用的回调函数。这个例子中，我们对于控件的点击事件并不关心，所以置为空。
* `setDisabledState(isDisabled: boolean)` -- 这个方法是可选的，顾名思义它是用来设置 `disabled` 状态的。

你可能观察到了，我们在装饰器中的 `providers` 中使用了一些陌生的关键字，比如 `forwardRef` 和 `multi`

首先来看 `forwardRef` -- 官方文档的定义是 **“允许引用一个尚未定义的引用”** ，这么说似乎还是懵懵懂懂的，让我们来举个例子

```ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-root',
  template: '<h1> {{ hello }}, World! </h1>'
})
class AppComponent {
  hello: string;

  constructor(helloService: HelloService) {
    this.hello = helloService.getHello();
  }
}

class HelloService {
  getHello () {
    return 'Hello';
  }
}
```

如果你试图运行这段代码，会发现这段代码根本无法跑起来，在 `console` 中会看到如下错误信息

```log
Uncaught Error: Can't resolve all parameters for AppComponent: (?).
```

这是由于在 `AppComponent` 的构造函数中，我们注入了 `HelloService` ，但是 `HelloService` 在此时还未定义。如果我们把 `HelloService` 移动到 `AppComponent` 的上方就没有错误了。天啊，难道在 Angular 中只能把类定义写在引用之前码？当然不是了， `forwardRef` 来拯救，像下面这样改造后就没问题了。

```ts
import {Component, Inject, forwardRef} from '@angular/core';

@Component({
  selector: 'app-root',
  template: '<h1> {{ hello }}, World! </h1>'
})
class AppComponent {
  hello: string;

  constructor(@Inject(forwardRef(() => HelloService)) helloService) {
    this.hello = helloService.getHello();
  }
}

class HelloService {
  getHello () {
    return 'Hello';
  }
}
```

再说回到我们的例子，我们是自己引用自己，所以当然得用 `forwardRef` 。那么接下来这个 `multi` 又是什么鬼？

先简单的复习一下依赖注入，一般情况下，同样的 token 只能对应一个依赖，如果使用同样的 token ，最后注册的那个胜出。

```ts
class Example { }
class AnotherExample { }

let injector = Injector.create([
  { provide: Example },
  { provide: Example, useClass: AnotherExample }
]);

let example = injector.get(Example);
// 一般情况下，同样的 token ，最后注册的那个胜出，
// 这里是 AnotherExample 胜出
```

但有些情况下，我们需要一个 token 对应多个依赖，比如验证器，我们会为同一个控件指定多个验证器，既有内建的也有外部自定义的，这种情况下，我们就需要一个 token 对应多个依赖。所以设置 `multi` 为 `true` 就是告诉系统，我们使用了一个 token （这里是 `NG_VALIDATORS` ）对应多个依赖，我们声明的是其中之一。当你注册依赖时，把我加进去，谢谢合作。

```ts
providers: [
  provide: NG_VALIDATORS,
  useValue: (formControl) => {
    // validation happens here
  },
  multi: true
]
```

好了，这个自定义表单控件算是做完了，我们在注册表单组件中使用看看，现在看上去是不是很和谐，和其他的表单组件一模一样啊。把这段加到 `src/app/auth/components/register-form/register-form.component.html` 中去。

```html
<app-image-picker [useSvgIcon]="true" [cols]="6" [title]="'选择头像：'" [items]="avatars" formControlName="avatar">
</app-image-picker>
```

接下来看看组件的写法，我们还是通过注入 `FormBuilder` 的方式来快速构建表单。

```ts
import { Component, Input, Output, EventEmitter, OnInit } from '@angular/core';
import {
  FormBuilder,
  FormGroup,
  Validators,
  AsyncValidatorFn,
  AbstractControl
} from '@angular/forms';
import {
  usernamePattern,
  mobilePattern,
  emailPattern,
  humanNamePattern
} from '../../../utils/regex';
import * as _ from 'lodash';
import {
  usernameErrorMsg,
  emailErrorMsg,
  humanNameErrorMsg,
  mobileErrorMsg
} from '../../../utils/validate-errors';

@Component({
  selector: 'register-form',
  templateUrl: './register-form.component.html',
  styleUrls: ['./register-form.component.scss']
})
export class RegisterFormComponent implements OnInit {
  @Input() avatars: string[] = [];
  @Input() title = '注册';
  @Input() subtitle = '注册成为会员体验全部功能';
  @Input() loginBtnText = '已经注册？点击登录';
  @Input() forgotBtnText = '忘记密码？';
  @Input() usernameValidator: AsyncValidatorFn;
  @Input() emailValidator: AsyncValidatorFn;
  @Input() mobileValidator: AsyncValidatorFn;
  @Output() submitEvent = new EventEmitter();
  form: FormGroup;
  private readonly avatarName = 'avatars';
  constructor(private fb: FormBuilder) {}

  ngOnInit(): void {
    this.avatars = _.range(1, 16)
      .map(i => `${this.avatarName}:svg-${i}`)
      .reduce((r: string[], x: string) => [...r, x], []);
    this.form = this.fb.group({
      login: [
        '',
        [
          Validators.required,
          Validators.minLength(3),
          Validators.maxLength(50),
          Validators.pattern(usernamePattern)
        ],
        this.usernameValidator
      ],
      mobile: [
        '',
        [Validators.required, Validators.pattern(mobilePattern)],
        this.mobileValidator
      ],
      email: [
        '',
        [Validators.required, Validators.pattern(emailPattern)],
        this.emailValidator
      ],
      name: [
        '',
        [
          Validators.required,
          Validators.minLength(2),
          Validators.maxLength(50),
          Validators.pattern(humanNamePattern)
        ]
      ],
      avatar: [],
      passwords: this.fb.group(
        {
          password: [
            '',
            [
              Validators.required,
              Validators.minLength(8),
              Validators.maxLength(20)
            ]
          ],
          repeat: ['', Validators.required]
        },
        { validator: this.matchPassword }
      )
    });
  }

  matchPassword(c: AbstractControl) {
    const password = c.get('password').value;
    const repeat = c.get('repeat').value;
    if (password !== repeat) {
      c.get('repeat').setErrors({ notMatchPassword: true });
      return { notMatchPassword: true };
    } else {
      return null;
    }
  }

  submit({ valid, value }: FormGroup, ev: Event) {
    if (!valid) {
      return;
    }
    this.submitEvent.emit(value);
  }

  get nameErrors() {
    const name = this.form.get('name');
    if (!name) {
      return '';
    }
    return humanNameErrorMsg(name);
  }

  get usernameErrors() {
    const username = this.form.get('username');
    if (!username) {
      return '';
    }
    return usernameErrorMsg(username);
  }

  get emailErrors() {
    const email = this.form.get('email');
    if (!email) {
      return '';
    }
    return emailErrorMsg(email);
  }

  get mobileErrors() {
    const mobile = this.form.get('mobile');
    if (!mobile) {
      return '';
    }
    return mobileErrorMsg(mobile);
  }

  get passwordErrors() {
    const password = this.form.get('passwords').get('password');
    if (!password) {
      return '';
    }
    return password.hasError('required')
      ? '密码为必填项'
      : password.hasError('minlength')
        ? `不能少于 ${password.errors['minlength'].requiredLength} 个字符`
        : password.hasError('maxlength')
          ? `不能超过 ${password.errors['maxlength'].requiredLength} 个字符`
          : '';
  }

  get repeatErrors() {
    const repeat = this.form.get('passwords').get('repeat');
    if (!repeat) {
      return '';
    }
    return repeat.hasError('required')
      ? '密码为必填项'
      : repeat.hasError('notMatchPassword') ? `密码不匹配` : '';
  }
}
```

这里面需要注意的有以下几个地方：

* 使用 `lodash` 快速创建一个 `range` 数组，用来构建图片的数据。使用 `lodash` 需要安装 `yarn add lodash` 以及其类型 `yarn add @types/lodash --dev`
* 对正则表达式类型的验证器，我们没有把正则表达式具体写在这个组件中，而是放在了 `src/app/utils/regex.ts` 中，这样其他需要此类正则表达式的类引用就好，也方便我们后期对正则表达式的更改和优化。
* 对于错误的显示处理，我们把对 `name` 、 `username` 、 `mobile` 和 `email` 的都抽取出去做了专门的函数，位于 `src/app/utils/validate-errors.ts`，这是由于这几个错误显示不只在这个地方会用到。
* 对于涉及到两个控件的验证器，我们在这个例子中的 `matchPassword` 采用的是对 `FormGroup` 构造验证器，然后在验证器中使用 `FormGroup` 的 `get` 方法得到对应的 `FormControl` 进行数据验证。

此外，我们为 `username` 、 `mobile` 和 `email` 等需要验证数据唯一性的控件预留了异步验证器。为什么需要异步验证呢？因为数据的唯一性需要到后端数据库进行查找比对，而连接后端是一个异步过程，这时我们就需要使用异步验证。再有这个组件是一个笨组件，所以我们把这几个异步验证器设置为输入型属性（对的，函数也可以作为属性），留给父组件使用这个组件时设置。

```ts
@Input() usernameValidator: AsyncValidatorFn;
@Input() emailValidator: AsyncValidatorFn;
@Input() mobileValidator: AsyncValidatorFn;
```

我们还遗留了“忘记密码”组件，这个组件比较复杂，放到下一节来讲。
