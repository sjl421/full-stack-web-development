# Angular 基础概念

Angular 中经常提到的几个概念：依赖性注入、模块、组件、指令、管道、模板驱动型表单和响应式表单。这些概念听上去令人头大，加上官网的解释又相对晦涩一些，导致很多人觉得 Angular 的学习门槛好高。但其实这些概念并不复杂，我们就逐一来揭开它们的面纱。

## 依赖性注入

### 什么是依赖性注入？

依赖性注入（ Dependency Injection ）其实不是 Angular 独有的概念，这是一个已经存在很长时间的设计模式，也可以叫做控制反转 （ Inverse of Control ）。我们从下面这个简单的代码片段入手来看看什么是依赖性注入以及为什么要使用依赖性注入。

```js
class Person {
  constructor() {
    this.address = new Address('北京', '北京', '朝阳区', 'xx街xx号');
    this.id = Id.getInstance(ID_TYPES.IDCARD);
  }
}
```

上面的代码中，我们在 `Person` 这个类的构造函数中初始化了我们构建 `Person` 所需要的依赖类： `Address` 和 `Id` ，其中 `Address` 是个人的地址对象，而 `Id` 是个人身份对象。这段代码的问题在于除了引入了内部所需的依赖之外， **它知道了这些依赖创建的细节** ，比如它知道 `Address` 的构造函数需要的参数（省、市、区和街道地址）和这些参数的顺序，它还知道 `Id` 的工厂方法和其参数（取得身份证类型的 `Id` ）。

但这样做的问题究竟是什么呢？首先这样的代码是非常难以进行单元测试的，因为在测试的时候我们往往需要构造一些不同的测试场景（比如我们想传入护照类型的 `Id` ），但这种写法导致你没办法改变其行为。其次，我们在代码的可维护性和扩展性方面有了很大的障碍，设想一下如果我们改变了 `Address` 的构造函数或 `Id` 的工厂方法的话，我们不得不去更改 `Person` 类。一个类还好，但如果几十个类都依赖 `Address` 或 `Person` 的话，这会造成多大的麻烦？

那么解决的方法呢？也很简单，那就是我们把 `Person` 的构造改造一下：

```js
class Person {
  constructor(address, id) {
    this.address = address;
    this.id = id;
  }
}
```

我们在构造中接受已经创建的 `Address` 和 `Id` 对象，这样在这段代码中就没有任何关于它们的具体实现了。换句话说，我们把创建这些依赖性的职责向上一级传递了出去（噗~~推卸责任啊）。现在我们在生产代码中可以这样构造 `Person` ：

```js
const person = new Person(
  new Address('北京', '北京', '朝阳区', 'xx街xx号'),
  Id.getInstance(ID_TYPES.IDCARD)
);
```

而在测试时，可以方便的构造各种场景，比如我们将地区改为辽宁：

```js
const person = new Person(
  new Address('辽宁', '沈阳', '和平区', 'xx街xx号'),
  Id.getInstance(ID_TYPES.PASSPORT)
);
```

其实这就是依赖性注入了，这个概念是不是很简单？但有的同学问了，那上一级要是单元测试不还是有问题吗？是的，如果上一级需要测试，就得『推卸责任』到再上一级了。这样一级一级的最后会推到最终的入口函数，但这也不是办法啊，而且靠人工维护也很容易出错，这时候就需要有一个依赖性注入的框架来解决了，这种框架一般叫做 DI 框架或者 IoC 框架。这种框架对于熟悉 Java 和 .Net 的同学不会陌生，鼎鼎大名的 Spring 最初就是一个这样的框架，当然现在功能丰富多了，远不止这个功能了。

### Angular 中的依赖性注入框架

Angular 中的依赖性注入框架主要包含下面几个角色：

* Injector（注入者）：使用 Injector 提供的 API 创建依赖的实例
* Provider（提供者）：Provider 告诉 Injector \*\*怎样\*\* 创建实例（比如我们上面提到的是通过某个构造函数还是工厂类创建等等）。Provider 接受一个令牌，然后把令牌映射到一个用于构建目标对象的工厂函数。
* Dependency（依赖）：依赖是一种 \*\*类型\*\* ，这个类型就是我们要创建的对象的类型。

![](/assets/chap_2_1_001.png)'

可能看到这里还是有些云里雾里，没关系，我们还是用例子来说明：

```js
import { ReflectiveInjector } from '@angular/core';
const injector = RelfectiveInjector.resolveAndCreate([
  // providers 数组定义了多个提供者，provide 属性定义令牌
  // useXXX 定义怎样创建的方法
  { provide: Person, useClass: Person },
  { provide: Address, useFactory: () => {
        if(env.testing)
            return new Address('辽宁', '沈阳', '和平区', 'xx街xx号');
        return new Address('北京', '北京', '朝阳区', 'xx街xx号');
    }
  },
  { provide: Id, useFactory: (type) => {
        if(type === ID_TYPES.PASSPORT)
            return Id.getInstance(ID_TYPES.PASSPORT, someparam);
        if(type === ID_TYPES.IDCARD)
            return Id.getInstance(ID_TYPES.IDCARD);
        return Id.getDefaultInstance();
    }
  }
]);

class Person {
  // 通过 @Inject 修饰器告诉 DI 这个参数需要什么样类型的对象
  // 请在 injector 中帮我找到并注入到对应参数中
  constructor(@Inject(Address) address, @Inject(Id) id) {
    // 省略
  }
}

// 通过 injector 得到对象
const person = injector.get(Person);
```

上述代码中，Angular 提供了 `RelfectiveInjector` 来解析和创建依赖的对象，你可以看到我们把这个应用中需要的 `Person` 、 `Id` 和 `Address` 都放在里面了。谁需要这些对象就可以向 `injector` 请求，比如： `injector.get(Person)` ，当然也可以 `injector.get(Address)` 等等。可以把它理解成一个依赖性的池子，想要什么就取就好了。

但是问题来了，首先 `injector` 怎么知道如何创建你需要的对象呢？这个是靠 `Provider` 定义的，在刚刚的 `RelfectiveInjector.resolveAndCreate()` 中我们发现它是接受一个数组作为参数，这个数组就是一个 `Provider` 的数组。`Provider` 最常见的属性有两个。第一个是 `provide` ，这个属性其实定义的是令牌，令牌的作用是让框架知道你要找的依赖是哪个然后就可以在 `useXXX` 这个属性定义的构建方式中将你需要的对象构建出来了。

那么 `constructor(@Inject(Address) address, @Inject(Id) id)` 这句怎么理解呢？由于我们在 `const person = injector.get(Person);` 想取得 Person ，但 Person 又需要两个依赖参数： `address` 和 `id` 。 `@Inject(Address) address` 是告诉框架我需要的是一个令牌为 `Address` 的对象，这样框架就又到 `injector` 中寻找令牌为 `Address` 对应的工厂函数，通过工厂函数构造好对象后又把对象赋值到 `address` 。

由于这里我们是用对象的类型来做令牌，上面的注入代码也可以写成下面的样子。利用 `Typescript` 的类型定义，框架看到有依赖的参数就会去 `Injector` 中寻找令牌为该类型的工厂函数。

```js
class Person {
  constructor(address: Address, id: Id) {
    // 省略
  }
}
```

而对于令牌为类型的并且是 `useClass` 的这种形式，由于前后都一样，对于这种 Provider 我们有一个语法糖：可以直接写成 `{ Person }` ，而不用完整的写成 `{ provide: Person, useClass: Person }` 这种形式。当然还要注意 `Token` 不一定非得是某个类的类型，也可以是字符串， Angular 中还有 `InjectionToken` 用于创建一个可以避免重名的 Token。

那么其实除了 `useClass` 和 `useFactory` ，我们还可以使用 `useValue` 来提供一些简单数据结构，比如我们可能希望把系统的 API 基础信息配置通过这种形式让所有想调用 API 的类都可以注入。如下面的例子中，基础配置就是一个简单的对象，里面有多个属性，这种情况用 `useValue` 即可。

```js
{
  provide: 'BASE_CONFIG',
  useValue: {
    uri: 'https://dev.local/1.1',
    apiSecret: 'blablabla',
    apiKey: 'nahnahnah'
  }
}
```

### 依赖性注入进阶

可能你注意到，上面提到的依赖性注入有一个特点，那就是需要注入的参数如果在 `Injector` 中找不到对应的依赖，那么就会发生异常了。但确实有些时候我们是需要这样的特性：该依赖是可选的，如果有我们就这么做，如果没有就那样做。遇到这种情况怎么办呢？

Angular 提供了一个非常贴心的 `@Optional` 修饰器，这个修饰器用来告诉框架后面的参数需要一个可选的依赖。

```js
constructor(@Optional(ThirdPartyLibrary) lib) {
    if (!lib) {
    // 如果该依赖不存在的情况
    }
}
```

需要注意的是，Angular 的 DI 框架创建的对象都是单件（ Singleton ）的，那么如果我们需要每次都创建一个新对象怎么破呢？我们有两个选择，第一种：在 Provider 中返回工厂而不是对象，像下面例子这样：

```js
  {
    provide: Address,
    useFactory: () => {
        // 注意：这里返回的是工厂，而不是对象
        return () => {
            if(env.testing)
                return new Address('辽宁', '沈阳', '和平区', 'xx街xx号');
            return new Address('北京', '北京', '朝阳区', 'xx街xx号');
        }
    }
  }
```

第二种：我们创建一个 `child injector` （子注入者）： `Injector.resolveAndCreateChild()`

```js
const injector = ReflectiveInjector.resolveAndCreate([Person]);
const childInjector = injector.resolveAndCreateChild([Person]);
// 此时父 Injector 和子 Injector 得到的 Person 对象是不同的
injector.get(Person) !== childInjector.get(Person);
```

而且子 Injector 还有一个特性：如果在 `childInjector` 中找不到令牌对应的工厂，它会去父 `Injector` 中寻找。换句话说，这父子关系（多重的）是构成了一棵依赖树，框架会从最下面的子 `Injector` 开始寻找，一直找到最上面的父 `Injector`。看到这里相信你就知道为什么父组件声明的 `providers` 对于子组件是可见的，因为子组件中在自己 `constructor` 中如果发现有找不到的依赖就会到父组件中去找。

在实际的 Angular 应用中我们其实很少会直接显式使用 `Injector` 去完成注入，而是在对应的模块、组件等的元数据中提供 `providers` 即可，这是由于 Angular 框架帮我们完成了这部分代码，它们其实在元数据配置后由框架放入 `Injector` 中了。

## 组件

组件是 Angular 应用中最基础的 UI 构建元素，从 UI 界面上可以暂时这么简单理解，一个组件就是渲染后的 HTML 页面中的某一块区域的元素、样式以及事件的集合。一个 Angular 的应用其实就是一棵组件树。组件其实是指令的一个子集，但不同的是，组件拥有自己的模板。任何一个组件都必须从属于一个模块，即在模块的 `declarations` 中声明。

`@Component` 修饰符用于标识一个类是 Angular 组件，和模块类似的也有很多元数据属性，一个典型的组件如下面代码所示

```js
@Component({
  // 其他组件调用时使用 <app-login></app-login> 标签
  // selector 可以想象成一个自定义 HTML 标签
  selector: 'app-login',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <input type="text" placeholder="您的Email">
    <input type="password" placeholder="您的密码">
  `,
  styleUrls: ['./login.component.scss']
})
export class LoginComponent implements OnInit {
    constructor(){
        // 省略
    }

    ngOnInit(){
        // 省略
    }
}
```

组件中支持的常见元数据如下：

* animations：动画定义
* changeDetection：设置脏值检测的策略
* encapsulation：样式的封装策略
* entryComponents：在此视图中需要动态加载的组件列表
* providers - 对此组件及其子组件可见的 Provider 列表
* selector - 用于定位此组件的选择器
* styleUrls：外部样式 URL 列表
* styles - 内联的样式
* template - 内联的模板
* templateUrl - 指向外部的模板文件 URL
* viewProviders：仅为此组件以及其视图上的子元素可见的 Provider 列表

组件有生命周期的概念：Angular 创建、渲染控件；创建、渲染子控件；当数据绑定属性改变时做检查；在把控件移除 DOM 之前销毁控件等等。Angular 提供生命周期的“钩子”（ Hook ）以便于开发者可以得到这些关键过程的数据以及在这些过程中做出响应的能力。这些函数和顺序可参见下表。上面的例子中我们监听组件的 OnInit 事件。

## 指令

Angular 中的指令分成三种：结构型（ Structural ）指令和属性型（ Attribute ）指令，还有一种是什么呢？就是 `Component` ，组件本身就是一个带模板的、有更多生命周期 Hook 的指令。组件这里就不做讨论了，我们主要看前两种，它们大致的形式可以理解成类似于 HTML 中元素的属性，比如 `<input type="text>"` 中的 `type`，指令和这个表现形式差不多。

下面的 `*ngIf` 就是一个典型的结构型指令，结构型指令一般前方都有一个 `*` ，用于控制 HTML 的布局，它们一般会生成或调整一些 DOM 元素，结构型指令没有绑定的概念，也就不会在指令外面发现 `()` 或 `[]` 。

```html
<a *ngIf="user.login">退出</a>
```

那么为什么会有这个看起来这么奇怪的 `*` 呢？它的作用是什么呢？加了这个 `*` ，指令会把自己所在的元素包装在一对 `<ng-template></ng-template>` 的模板之中，也就是说上面的代码相当于下面的形式，注意此时 `*` 没有了，所以其实这个 `*` 也是一个语法糖，写一个 `*` 省去了写个嵌套的 `<ng-template></ng-template>`。

```html
<ng-template ngIf="user.login">
  <a>退出</a>
</ng-template>
```

把指令所在的元素包含在模板中的好处就是，你可以取得完全的控制权，我们可以根据需要对此模板内的元素显示、隐藏、增加或删除等等。对于这个模板，我们在指令内部可以通过 `TemplateRef` 得到其引用。知道了这些，我们可以一起来看看 `ngIf` 这个指令的源码，你就会明白为什么根据表达式的真假，当前元素就可以显示或不显示了：

```js
import {Directive, Input, TemplateRef, ViewContainerRef} from '@angular/core';
@Directive({selector: '[ngIf]'})
export class NgIf {
  private _hasView = false;

  constructor(private _viewContainer: ViewContainerRef, private _template: TemplateRef<Object>) {}

  @Input()
  set ngIf(condition: any) {
    if (condition && !this._hasView) {
      this._hasView = true;
      // 如果条件为真，创建该模板元素
      this._viewContainer.createEmbeddedView(this._template);
    } else if (!condition && this._hasView) {
      this._hasView = false;
      // 否则清空视图
      this._viewContainer.clear();
    }
  }
}
```

常见的内建结构型指令除了 `ngIf` 还有 `ngFor` 和 `ngSwitchCase` 等。

属性型指令是改变 DOM 元素的外观或行为的指令，而常见的内建属性型指令有 `ngModel` 和 `ngClass` ，其中 `ngModel` 我们会在后面的模板驱动型表单中讲解，这里看一下 `ngClass` 。很多时候，我们希望动态的改变元素或组件的样式，这个时候 `ngClass` 就派上用场了。 `ngClass` 可以接收很多种类型的参数，包括以空格分隔的字符串、数组以及像下面这样的对象。对象的 `key` 是 CSS 的类名，`value` 是一个布尔值。个人以为这种对象的形式最为灵活，因为你可以在程序中动态控制各个 CSS 类的 `true` 或 `false`，即是否使用。

```html
<some-element [ngClass]="{'first': true, 'second': true, 'third': false}">...</some-element>
```

属性型指令经常会在 DOM 元素属性的外面套上 `[]` 或 `()` ，比如上面的 `[ngClass]="xxx"` ， `[]` 是说 `"xxx"` 是一个表达式（或者对象或变量），请将这个表达式的值赋值给 `ngClass` 。但如果不加 `[]` ，那么 Angular 会认为你要把 `"xxx"` 这个**字符串**赋值给 `ngClass` 了。那么 `()` 呢？这个是用来绑定事件的，有时候指令或组件会有事件（也就是输出型参数），这个时候就需要用 `()` 绑定这个事件，用来监听处理。

## 管道

管道是一种在模板中使用的快速方便进行数据的变换的快捷方式，使用上来说是 `| 管道名称: "参数"` 这样的形式。 这个特性可以让我们很快的将数据在界面上以我们想要的格式输出出来。看一个小例子：

```js
import { Component, OnDestroy } from '@angular/core';

@Component({
    selector: 'app-playground',
    template: `
    <p>Without Pipe: Today is {{ birthday }} </p>
    <p>With Pipe: Today is {{ birthday | date:"MM/dd/yy" }} </p>
    `,
    styleUrls: ['./playground.component.css']
})
export class PlaygroundComponent {
    birthday = new Date();
    constructor() { }
}
```

![Angular 中的管道](/assets/chap_2_1_002.png)'

## 模块

首先什么是模块？简单来说就是把一些功能组织到一起完成一个相对独立的事情。打个比方，常见的应用都有登录、注册、忘记密码这些功能，但这些功能在应用中其他的地方使用的场景多吗？好像是联系不大，这块功能是相对独立的，那么也就是说我们可以把这些登录注册相关的功能封装在一个模块中（这里我们把这个模块起个名字，叫 `AuthModule` 吧），然后提供一些必要的接口和组件给外部，系统的其他部分想调用时可以导入这个模块就行了，无需关心这些功能是怎么实现的。比如说我们在登录模块中增加一个微信登录的方式，我们就只需要在这个 `AuthModule` 中改动即可，其他模块不需要进行更改。

可能有些熟悉 Javascript 的同学还会问，Javascript 中已经有模块的概念了，为什么还要再搞出一个呢？这是由于 Javascript 中的模块概念是每一个文件是一个模块，但是当我们有很多文件时，比如我们有 10 个文件，而这时我们需要增加 5 个文件，这两个文件呢是需要依赖这 10 个文件的，那么我们就需要在这 5 个文件中 `import` 那 10 个文件。当工程大起来之后，你想想这个工作量和代码维护性。所以用模块形式组织代码是大工程的必要步骤。

在 Angular 中，我们会使用一个 `@NgModule` 的修饰符来标识一个类是模块，在这个修饰符中，我们会定义一个模块的元数据：

* declarations：用于列出属于此模块的组件、指令和管道
* imports：用于导入该模块依赖的其他模块
* exports：如果有想让其他模块使用的组件、指令和管道，在此列出
* providers：列出提供给模块内部用于注入的资源，通常是服务
* entryComponents：列出进入模块就要实例化的组件，用于动态加载
* bootstrap：只有根模块有此项，用于指定应用启动后的根组件

一个典型的模块在 Angular 中的定义是长成下面的样子：

```js
@NgModule({
    declarations: [ // 此模块包含的组件/指令/管道都列在这个数组中
        HomeComponent,
        AComponent, BComponent,
        CDirective,
        DPipe,
    ],
    imports: [ // 该模块所依赖的其他模块在这个数组中列出进行导入
        CommonModule,
        ThridPartyModule,
    ],
    exports: [ // 导出给外部模块可以使用的组件/指令/管道等
        AComponent
    ]
    providers: [ // 该模块要提供给模块内部注入使用的类库，一般是服务
        MyOwnService,
    ],
    entryComponents: [ // 对于有些组件需要在进入模块后就创建出来，比如对话框等
        HomeComponent
    ]
})
export class MyModule { }
```

任何一个 Angular 应用至少有一个模块：根模块。习惯上我们把根模块命名为 `AppModule`。另外需要注意一点，模块是一个类，尽管很多时候我们都没有为这个类写具体的方法，但这不代表这个类不能写具体的内部的成员方法。事实上，有一些情况你是非得写方法才可以达成目标的，比如：如果你希望模块被导入后要进行一些初始化工作：

```js
@NgModule({
    // 省略
})
export class MyModule {
    constructor(){
        doInitWork();
    }
    void doInitWork(){
      // 省略
    }
}
```

我们看到内建的模块比如 `HttpModule` 或者很多第三方模块，在导入的同时，其提供的 `Providers` 也就可以调用了，并不用显式的在我们的 `Module` 中的 `providers` 数组中列出。这其实也是利用了模块的构造或实例方法完成的。像下面的例子中，我们在模块类中提供一个静态方法返回模块和 `providers`，这样只要在根模块中的 `imports` 中写 `MyModule.forRoot()` 即可完成导入模块的同时将模块提供的 `providers` 也提供给根模块。注意一点，命名上习惯性的将只在根模块使用的方法命名为 `forRoot` 。

```js
@NgModule({
  // 省略
})
class MyModule {
  static forRoot() {
    return {
      ngModule: MyModule,
      providers: [ AuthService ]
    }
  }
}
```

## 模板驱动型表单

在企业应用开发时，表单是一个躲不过去的事情，和面向消费者的应用不同，企业领域的开发中，表单的使用量是惊人的。这些表单的处理其实是一个挺复杂的事情，比如有的是涉及到多个 Tab 的表单，有的是向导形式多个步骤的，各种复杂的验证逻辑和时不时需要弹出的对话框等等。

Angular 中提供两种类型的表单处理机制，一种叫模版驱动型（ Template Driven ）的表单，另一种叫模型驱动型表单（ Model Driven ），这后一种也叫响应式表单 （ Reactive Forms ），由于模版驱动中有一个 `ngModel` 的指令，容易和这里说的模型驱动混淆，所以在我们的文章中叫后一种说法：响应式表单。

模版驱动的表单和 AngularJS 对于表单的处理类似，把一些指令（比如 `ngModel` ）、数据值和行为约束（比如 `require` 、`minlength` 等等）绑定到模版中（模版就是组件元数据 `@Component` 中定义的那个 `template` ），这也是模版驱动这个叫法的来源。总体来说，这种类型的表单通过绑定把很多工作交给了模版。

### 模板驱动型表单的例子

还是用例子来说话，比如我们有一个用户注册的表单，用户名就是 `email` ，还需要填的信息有：住址、密码和重复密码。这个应该是比较常见的一个注册时需要的信息了。那么我们第一步来建立领域模型：

```typescript
// src/app/domain/index.ts
export interface User {
  // 新的用户id一般由服务器自动生成，所以可以为空，用 ? 标示
  id?: string;
  email: string;
  password: string;
  repeat: string;
  address: Address;
}

export interface Address {
  province: string; // 省份
  city: string; // 城市
  area: string; // 区县
  addr: string; // 详细地址
}
```

接下来我们建立模版文件，一个最简单的 HTML 模版，先不增加任何的绑定或事件处理：

```html
<!-- template-driven.component.html -->
<form novalidate>
  <label>
    <span>电子邮件地址</span>
    <input
      type="text"
      name="email"
      placeholder="请输入您的 email 地址">
  </label>
  <div>
    <label>
      <span>密码</span>
      <input
        type="password"
        name="password"
        placeholder="请输入您的密码">
    </label>
    <label>
      <span>确认密码</span>
      <input
        type="password"
        name="repeat"
        placeholder="请再次输入密码">
    </label>
  </div>
  <div >
    <label>
      <span>省份</span>
      <select name="province">
        <option value="">请选择省份</option>
      </select>
    </label>
    <label>
      <span>城市</span>
      <select name="city">
        <option value="">请选择城市</option>
      </select>
    </label>
    <label>
      <span>区县</span>
      <select name="area">
        <option value="">请选择区县</option>
      </select>
    </label>
    <label>
      <span>地址</span>
      <input type="text" name="addr">
    </label>
  </div>
  <button type="submit">注册</button>
</form>
```

渲染之后的效果就像下面这样：

![简单的Form](/assets/chap_2_1_003.png)

### 数据绑定

对于模版驱动型的表单处理，我们首先需要在对应的模块中引入 `FormsModule` ，这一点千万不要忘记了。

```typescript
import { NgModule } from '@angular/core';
import { CommonModule } from '@angular/common';
import { FormsModule } from "@angular/forms";
import { TemplateDrivenComponent } from './template-driven/template-driven.component';

@NgModule({
  imports: [
    CommonModule,
    FormsModule
  ],
  exports: [TemplateDrivenComponent],
  declarations: [TemplateDrivenComponent]
})
export class FormDemoModule { }

```

进行模版驱动类型的表单处理的一个必要步骤就是建立数据的双向绑定，那么我们需要在组件中建立一个类型为 `User` 的成员变量并赋初始值。

```typescript
// template-driven.component.ts
// 省略元数据和导入的类库信息
export class TemplateDrivenComponent implements OnInit {

  user: User = {
    email: '',
    password: '',
    repeat: '',
    address: {
      province: '',
      city: '',
      area: '',
      addr: ''
    }
  };
  // 省略其他部分
}
```

有了这样一个成员变量之后，我们在组件模版中就可以使用 `ngModel` 进行绑定了。

### 令人困惑的 `ngModel`

我们在 Angular 中可以使用三种形式的 `ngModel` 表达式： `ngModel` , `[ngModel]` 和 `[(ngModel)]`。但无论那种形式，如果你要使用 `ngModel` 就必须为该控件（比如下面的 `input` ）指定一个 `name` 属性，如果你忘记添加 `name` 的话，多半你会看到下面这样的错误：

```bash
ERROR Error: Uncaught (in promise): Error: If ngModel is used within a form tag, either the name attribute must be set or the form control must be defined as 'standalone' in ngModelOptions.
```

假如我们使用的是 ngModel ，没有任何中括号小括号的话，这代表着我们创建了一个 FormControl 的实例，这个实例将会跟踪值的变化、用户的交互、验证状态以及保持视图和领域对象的同步等工作。

```html
<input
  type="text"
  name="email"
  placeholder="请输入您的 email 地址"
  ngModel>
```

如果我们将这个控件放在一个 Form 表单中， `ngModel` 会自动将这个 FormControl 注册为 Form 的子控件。下面的例子中我们在 `<form>` 中加上了 `ngForm` 指令，声明这是一个 Angular 可识别的表单，而 `ngModel` 会将 `<input>` 注册成表单的子控件，这个子控件的名字就是 `email`，而且 `ngModel` 会基于这个子控件的值去绑定表单的的值，这也是为什么需要显式声明 `name` 的原因。

其实在我们导入 `FormsModule` 的时候，所有的 `<form>` 标签都会默认的被认为是一个 `NgForm` ，因此我们并不需要显式的在标签中写 `ngForm` 这个指令。

```html
<!-- ngForm 并不需要显示声明，任何 <form> 标签默认都是 ngForm -->
<form novalidate ngForm>
  <input
    type="text"
    name="email"
    placeholder="请输入您的 email 地址"
    ngModel>
</form>
```

这一切现在都是不可见的，所以大家可能还是有些困惑，那么下面我们将其“可视化”，这需要我们引用一下表单对象，所以我们使用 `#f="ngForm"` 以便我们可以在模版中输出表单的一些特性。

```html
<!-- 使用 # 把表单对象导出到 f 这个可引用变量中 -->
<form novalidate #f="ngForm">
  ...
</form>
<!-- 将表单的值以 JSON 形式输出 -->
{{f.value | json}}
```

这时如果我们在 email 中输入 `sss` ，可以看到下图的以 JSON 形式出现的表单值：

![控件的输入值同步到了表单的值中](/assets/chap_2_1_004.png)

那么接下来，我们看看 `[ngModel]` 有什么用？如果我们想给控件设置一个初始值怎么办呢，这时就需要进行一个单向绑定，方向是从组件到视图。我们可以做的是在初始化 `User` 的时候，将 `email` 属性设置成 `wang@163.com`

```typescript
user: User = {
    email: 'wang@163.com',
    ...
  };
```

而且在模版中使用 `[ngModel]="user.email"` 进行单向绑定，这个语法其实和普通的属性绑定是一样的，用中括号标示这是一个要进行数据绑定的属性，等号右边是需要绑定的值（这里是 `user.email` ）。那么我们就可以得到下面这样的输出了， `email` 的初始值被绑定成功！

![单向数据绑定](/assets/chap_2_1_005.png)

但上面的例子存在一个问题，数据的绑定是单向的，也就是说，在输入框进行输入的时候，我们的 `user` 的值不会随之改变的。为了更好的说明，我们将 `user` 和 表单的值同时输出

```html
<div>
  <span>user: </span> {{user | json}}
</div>
<div>
  <span>表单：</span> {{f.value | json}}
</div>
```

此时我们将默认的电子邮件改成 `wang@gmail.com` 的话，表单的值是改变了，但 `user` 并未改变。

![输入的值影响了表单，但不会影响领域对象](/assets/chap_2_1_006.png)

如果我们希望的是在输入时，这个输入的值也反向的影响我们的 `user` 对象的值的话，那就需要用到双向绑定了，也就是 `[(ngModel)]` 需要上场了。

![表单和领域对象的值保持了同步](/assets/chap_2_1_007.png)

无论如何，这个 `[()]` 表达真是很奇怪的样子，其实这个表达是一个语法糖。只要我们知道下面的两种写法是等价的，我们就会很清楚的理解了：用这个语法糖你就不用既写数据绑定又写事件绑定了。

```html
<input [(ngModel)]="user.email">
<input [ngModel]="user.email"` (ngModelChange)="user.email = $event">
```

如果我们仔细观察上面的输出的话，会发现一个问题： `user` 中是有一个嵌套对象 `address` 的，而表单中没有嵌套对象的。如果要实现表单中的结构和领域对象的结构一致的话，我们就得请出 `ngModelGroup` 了。`ngModelGroup` 会创建并绑定一个 FormGroup 到该 DOM 元素。 FormGroup 又是什么呢？简单来说，是一组 FormControl。

```html
  <!-- 使用 ngModelGroup 来创建并绑定 FormGroup  -->
  <div ngModelGroup="address">
    <label>
      <span>省份</span>
      <select name="province" (change)="onProvinceChange()" [(ngModel)]="user.address.province">
        <option value="">请选择省份</option>
        <option [value]="province" *ngFor="let province of provinces">{{province}}</option>
      </select>
    </label>
    <!-- 省略其他部分 -->
  </div>
```

这样的话，我们再来看一下输出，现在就完全一致了：

![表单和领域对象的结构也完全一致了](/assets/chap_2_1_008.png)

### 数据验证

模版驱动型的表单的验证也是主要由模版来处理的，在看怎么使用之前，需要界定一下验证规则：

* 三个必填项： `email`, `password` 和 `repeat`
* `email` 的形式需要符合电子邮件的标准
* `password` 和 `repeat` 必须一致

当然除了这几个规则，我们还希望在表单未验证通过时提交按钮是不可用的。

```html
<form novalidate #f="ngForm">
  <label>
    <span>电子邮件地址</span>
    <input
      type="text"
      name="email"
      placeholder="请输入您的 email 地址"
      [ngModel]="user.email"
      required
      pattern="([a-zA-Z0-9]+[_|_|.]?)*[a-zA-Z0-9]+@([a-zA-Z0-9]+[_|_|.]?)*[a-zA-Z0-9]+.[a-zA-Z]{2,4}">
  </label>
  <div>
    <label>
      <span>密码</span>
      <input
        type="password"
        name="password"
        placeholder="请输入您的密码"
        [(ngModel)]="user.password"
        required
        minlength="8">
    </label>
    <label>
      <span>确认密码</span>
      <input
        type="password"
        name="repeat"
        placeholder="请再次输入密码"
        [(ngModel)]="user.repeat"
        required
        minlength="8">
    </label>
  </div>
  <!-- 省略其他部分 -->
  <button type="submit" [disabled]="f.invalid">注册</button>
</form>
<div>

```

Angular 中有几种内建支持的验证器（ Validators ）

* required - 需要 FormControl 有非空值
* minlength - 需要 FormControl 有最小长度的值
* maxlength - 需要 FormControl 有最大长度的值
* pattern - 需要 FormControl 的值可以匹配正则表达式
* email - 内建的电子邮件的验证器

如果我们想看到结果的话，我们可以在模版中加上下面的代码，将错误以 JSON 形式输出即可。

```html
<div>
  <span>email 验证：</span> {{f.controls.email?.errors | json}}
</div>
```

我们看到，如果不填电子邮件的话，错误的 JSON 是 `{"required": true}` ，这告诉我们目前有一个 `required` 的规则没有被满足。

![验证结果](/assets/chap_2_1_009.png)

当我们输入一个字母 `w` 之后，就会发现错误变成了下面的样子。这是因为我们对于 `email` 应用了多个规则，当必填项满足后，系统会继续检查其他验证结果。

```json
{
"pattern":
    {
        "requiredPattern": "^([a-zA-Z0-9]+[_|_|.]?)*[a-zA-Z0-9]+@([a-zA-Z0-9]+[_|_|.]?)*[a-zA-Z0-9]+.[a-zA-Z]{2,4}$",
        "actualValue": "w"
    }
}
```

通过几次实验，我们应该可以得出结论，当验证未通过时，验证器返回的是一个对象， key 为验证的规则（比如 required, minlength 等），value 为验证结果。如果验证通过，返回的是一个 `null` 。

知道这一点后，我们其实就可以做出验证出错的提示了，为了方便引用，我们还是导出 `ngModel` 到一个 `email` 引用，然后就可以访问这个 FormControl 的各个属性了：验证的状态（ valid/invalid ）、控件的状态（是否获得过焦点 -- touched/untouched，是否更改过内容 -- pristine/dirty 等）

```html
<label>
  <span>电子邮件地址</span>
  <input
    ...
    [ngModel]="user.email"
    #email="ngModel">
</label>
<div *ngIf="email.errors?.required && email.touched" class="error">
  email 是必填项
</div>
<div *ngIf="email.errors?.pattern && email.touched" class="error">
  email 格式不正确
</div>
```

内建的验证器对于两个密码比较的这种验证是不够的，那么这就需要我们自己定义一个验证器。对于响应式表单来说，会比较简单一些，但对于模版驱动的表单，这需要我们实现一个指令来使这个验证器更通用和更一致。因为我们希望实现的样子应该是和 `required`、`minlength` 等差不多的形式，比如下面这个样子 `validateEqual="repeat"`

```html
<div>
    <label>
      <span>密码</span>
      <input
        type="password"
        name="password"
        placeholder="请输入您的密码"
        [(ngModel)]="user.password"
        required
        minlength="8"
        validateEqual="repeat">
    </label>
    <label>
      <span>确认密码</span>
      <input
        type="password"
        name="repeat"
        placeholder="请再次输入密码"
        [(ngModel)]="user.repeat"
        required
        minlength="8">
    </label>
  </div>
```

那么要实现这种形式的验证的话，我们需要建立一个指令，而且这个指令应该实现 `Validator` 接口。一个基础的框架如下：

```typescript
import { Directive, forwardRef } from '@angular/core';
import { NG_VALIDATORS, Validator, AbstractControl } from '@angular/forms';

@Directive({
  selector: '[validateEqual][ngModel]',
  providers: [
    {
      provide: NG_VALIDATORS,
      useExisting: forwardRef(()=>RepeatValidatorDirective),
      multi: true
    }
  ]
})
export class RepeatValidatorDirective implements Validator{
  constructor() { }
  validate(c: AbstractControl): { [key: string]: any } {
    return null;
  }
}
```

我们还没有开始正式的写验证逻辑，但上面的框架已经出现了几个有意思的点：

1. Validator 接口要求必须实现的一个方法是 `validate(c: AbstractControl): ValidationErrors | null;` 。这个也就是我们前面提到的验证正确返回 `null` 否则返回一个对象，虽然没有严格的约束，但其 `key` 一般用于表示这个验证器的名字或者验证的规则名字，`value` 一般是失败的原因或验证结果。
2. 和组件类似，指令也有 `selector` 这个元数据，用于选择那个元素应用该指令，那么我们这里除了要求 DOM 元素应用 `validateEqual` 之外，还需要它是一个 `ngModel` 元素，这样它才是一个 `FormControl`，我们在 `validate` 的时候才是合法的。
3. 那么那个 `providers` 里面那些面目可憎的家伙又是干什么的呢？ Angular 对于在一个 `FormControl` 上执行验证器有一个内部机制： Angular 维护一个令牌为 `NG_VALIDATORS` 的 `multi provider`（简单来说，Angular 为一个单一令牌注入多个值的这种形式叫 `multi provider` ）。所有的内建验证器都是加到这个 `NG_VALIDATORS`  的令牌上的，因此在做验证时，Angular 是注入了 `NG_VALIDATORS` 的依赖，也就是所有的验证器，然后一个个的按顺序执行。因此我们这里也把自己加到这个 `NG_VALIDATORS` 中去。
4. 但如果我们直接写成 `useExisting: RepeatValidatorDirective` 会出现一个问题， `RepeatValidatorDirective` 还没有生成，你怎么能在元数据中使用呢？这就需要使用 `forwardRef` 来解决这个问题，它接受一个返回一个类的函数作为参数，但这个函数不会立即被调用，而是在该类声明后被调用，也就避免了 `undefined` 的状况。

下面我们就来实现这个验证逻辑，由于密码和确认密码有主从关系，并非完全的平行关系。也就是说，密码是一个基准对比对象，当密码改变时，我们不应该提示密码和确认密码不符，而是应该将错误放在确认密码中。所以我们给出另一个属性 `reverse`。

```typescript

export class RepeatValidatorDirective implements Validator{
  constructor(
    @Attribute('validateEqual') public validateEqual: string,
    @Attribute('reverse') public reverse: string) { }

  private get isReverse() {
    if (!this.reverse) return false;
    return this.reverse === 'true' ? true: false;
  }

  validate(c: AbstractControl): { [key: string]: any } {
    // 控件自身值
    let self = c.value;

    // 要对比的值，也就是在 validateEqual=“ctrlname” 的那个控件的值
    let target = c.root.get(this.validateEqual);

    // 不反向查询且值不相等
    if (target && self !== target.value && !this.isReverse) {
      return {
        validateEqual: true
      }
    }

    // 反向查询且值相等
    if (target && self === target.value && this.isReverse) {
        delete target.errors['validateEqual'];
        if (!Object.keys(target.errors).length) target.setErrors(null);
    }

    // 反向查询且值不相等
    if (target && self !== target.value && this.isReverse) {
        target.setErrors({
            validateEqual: true
        })
    }

    return null;
  }
}

```

这样改造后，我们的模版文件中对于密码和确认密码的验证器如下：

```html
<input
    type="password"
    name="password"
    placeholder="请输入您的密码"
    [(ngModel)]="user.password"
    #password="ngModel"
    required
    minlength="8"
    validateEqual="repeat"
    reverse="true">
<!-- 省略其他部分 -->
<input
    type="password"
    name="repeat"
    placeholder="请再次输入密码"
    [(ngModel)]="user.repeat"
    #repeat="ngModel"
    required
    minlength="8"
    validateEqual="password"
    reverse="false">
```

![完成后的验证错误提示](/assets/chap_2_1_010.png)

### 表单的提交

表单的提交比较简单，绑定表单的 `ngSubmit` 事件即可

```html
<form novalidate #f="ngForm" (ngSubmit)="onSubmit(f, $event)">
```

但需要注意的一点是，button如果不指定类型的话，会被当做 `type="submit"`，所以当按钮不是进行提交表单的话，需要显式指定 `type="button"` 。而且如果遇到点击提交按钮页面刷新的情况的话，意味着默认的表单提交事件引起了浏览器的刷新，这种时候需要阻止事件冒泡。

```typescript
onSubmit({value, valid}, event: Event){
  if(valid){
    console.log(value);
  }
  event.preventDefault();
}
```

## 响应式表单

响应式表单乍一看还是很像模板驱动型表单的，但响应式表单需要引入一个不同的模块： `ReactiveFormsModule` 而不是 `FormsModule`

```typescript
import {ReactiveFormsModule} from "@angular/forms";
@NgModule({
  // 省略其他
    imports: [..., ReactiveFormsModule],
  // 省略其他
})
// 省略其他
```

### 与模板驱动型表单的区别

接下来我们还是利用前面的例子，用响应式表单的要求改写一下：

```html
<form [formGroup]="user" (ngSubmit)="onSubmit(user)">
  <label>
    <span>电子邮件地址</span>
    <input type="text" formControlName="email" placeholder="请输入您的 email 地址">
  </label>
  <div *ngIf="user.get('email').hasError('required') && user.get('email').touched" class="error">
    email 是必填项
  </div>
  <div *ngIf="user.get('email').hasError('pattern') && user.get('email').touched" class="error">
    email 格式不正确
  </div>
  <div>
    <label>
      <span>密码</span>
      <input type="password" formControlName="password" placeholder="请输入您的密码">
    </label>
    <div *ngIf="user.get('password').hasError('required') && user.get('password').touched" class="error">
      密码是必填项
    </div>
    <label>
      <span>确认密码</span>
      <input type="password" formControlName="repeat" placeholder="请再次输入密码">
    </label>
    <div *ngIf="user.get('repeat').hasError('required') && user.get('repeat').touched" class="error">
      确认密码是必填项
    </div>
    <div *ngIf="user.hasError('validateEqual') && user.get('repeat').touched" class="error">
      确认密码和密码不一致
    </div>
  </div>
  <div formGroupName="address">
    <label>
      <span>省份</span>
      <select formControlName="province">
        <option value="">请选择省份</option>
        <option [value]="province" *ngFor="let province of provinces">{{province}}</option>
      </select>
    </label>
    <label>
      <span>城市</span>
      <select formControlName="city">
        <option value="">请选择城市</option>
        <option [value]="city" *ngFor="let city of (cities$ | async)">{{city}}</option>
      </select>
    </label>
    <label>
      <span>区县</span>
      <select formControlName="area">
        <option value="">请选择区县</option>
        <option [value]="area" *ngFor="let area of (areas$ | async)">{{area}}</option>
      </select>
    </label>
    <label>
      <span>地址</span>
      <input type="text" formControlName="addr">
    </label>
  </div>
  <button type="submit" [disabled]="user.invalid">注册</button>
</form>
```

这段代码和模板驱动型表单的那段看起来差不多，但是有几个区别：

* 表单多了一个指令 `[formGroup]="user"`
* 去掉了对表单的引用 `#f="ngForm"`
* 每个控件多了一个 `formControlName`
* 但同时每个控件也去掉了验证条件，比如 `required`、`minlength` 等
* 在地址分组中用 `formGroupName="address"` 替代了 `ngModelGroup="address"`

模板上的区别大概就这样了，接下来我们来看看组件的区别：

```javascript
import { Component, OnInit } from '@angular/core';
import { FormControl, FormGroup, Validators } from "@angular/forms";
@Component({
  selector: 'app-model-driven',
  templateUrl: './model-driven.component.html',
  styleUrls: ['./model-driven.component.css']
})
export class ModelDrivenComponent implements OnInit {

  user: FormGroup;

  ngOnInit() {
    // 初始化表单
    this.user = new FormGroup({
      email: new FormControl('', [Validators.required, Validators.pattern(/([a-zA-Z0-9]+[_|_|.]?)*[a-zA-Z0-9]+@([a-zA-Z0-9]+[_|_|.]?)*[a-zA-Z0-9]+.[a-zA-Z]{2,4}/)]),
      password: new FormControl('', [Validators.required]),
      repeat: new FormControl('', [Validators.required]),
      address: new FormGroup({
        province: new FormControl(''),
        city: new FormControl(''),
        area: new FormControl(''),
        addr: new FormControl('')
      })
    });
  }
  onSubmit({value, valid}){
    if(!valid) return;
    console.log(JSON.stringify(value));
  }
}
```

从上面的代码中我们可以看到，这里的表单（ `FormGroup` ）是由一系列的表单控件（ `FormControl` ）构成的。其实 `FormGroup` 的构造函数接受的是三个参数： `controls`（表单控件『数组』，其实不是数组，是一个类似字典的对象） 、 `validator`（验证器） 和  `asyncValidator`（异步验证器） ，其中只有 `controls` 数组是必须的参数，后两个都是可选参数。

```ts
// FormGroup 的构造函数
constructor(
  controls: {
    [key: string]: AbstractControl;
  },
  validator?: ValidatorFn,
  asyncValidator?: AsyncValidatorFn
)
```

我们上面的代码中就没有使用验证器和异步验证器的可选参数，而且注意到我们提供 `controls` 的方式是，一个 `key` 对应一个 `FormControl` 。比如下面的 `key` 是 `password`，对应的值是 `new FormControl('', [Validators.required])` 。这个 `key` 对应的就是模板中的 `formControlName` 的值，我们模板代码中设置了 `formControlName="password"` ，而表单控件会根据这个 `password`  的控件名来跟踪实际的渲染出的表单页面上的控件（比如 `<input formcontrolname="password">`）的值和验证状态。

```javascript
password: new FormControl('', [Validators.required])
```

那么可以看出，这个表单控件的构造函数同样也接受三个**可选**参数，分别是：控件初始值（ `formState` ）、控件验证器或验证器数组（ `validator` ）和控件异步验证器或异步验证器数组（ `asyncValidator` ）。上面的那行代码中，初始值为空字符串，验证器是『必选』，而异步验证器我们没有提供。

```ts
// FormControl 的构造函数
constructor(
  formState?: any, // 控件初始值
  validator?: ValidatorFn | ValidatorFn[], // 控件验证器或验证器数组
  asyncValidator?: AsyncValidatorFn | AsyncValidatorFn[] // 控件异步验证器或异步验证器数组
)
```

由此可以看出，响应式表单区别于模板驱动型表单的的主要特点在于：是由组件类去创建、维护和跟踪表单的变化，而不是依赖模板。

那么我们是否在响应式表单中还可以使用 `ngModel` 呢？当然可以，但这样的话表单的值会在两个不同的位置存储了： `ngModel` 绑定的对象和 `FormGroup` ，这个在设计上我们一般是要避免的，也就是说尽管可以这么做，但我们不建议这么做。

### 使用 FormBuilder 快速构建表单

上面的表单构造起来虽然也不算太麻烦，但是在表单项目逐渐多起来之后还是一个挺麻烦的工作，所以 Angular 提供了一种快捷构造表单的方式 -- 使用 FormBuilder。

```ts
import { Component, OnInit } from '@angular/core';
import { FormBuilder, FormGroup, Validators } from "@angular/forms";
@Component({
  selector: 'app-model-driven',
  templateUrl: './model-driven.component.html',
  styleUrls: ['./model-driven.component.css']
})
export class ModelDrivenComponent implements OnInit {

  user: FormGroup;

  constructor(private fb: FormBuilder) {
  }

  ngOnInit() {
    // 初始化表单
    this.user = this.fb.group({
      email: ['', [Validators.required, Validators.email]],
      password: ['', Validators.required],
      repeat: ['', Validators.required],
      address: this.fb.group({
        province: [],
        city: [],
        area: [],
        addr: []
      })
    });
  }
  // 省略其他部分
}
```

使用 FormBuilder 我们可以无需显式声明 FormControl 或 FormGroup 。 FormBuilder 提供三种类型的快速构造： `control` , `group` 和 `array` ，分别对应 FormControl, FormGroup 和 FormArray。 我们在表单中最常见的一种是通过 `group` 来初始化整个表单。上面的例子中，我们可以看到 `group` 接受一个字典对象作为参数，这个字典中的 key 就是这个 FormGroup 中 FormControl 的名字，值是一个数组，数组中的第一个值是控件的初始值，第二个是同步验证器的数组，第三个是异步验证器数组（第三个并未出现在我们的例子中）。这其实已经在隐性的使用 `FormBuilder.control` 了，可以参看下面的 FormBuilder 中的 `control` 函数定义，其实 FormBuilder 利用我们给出的值构造了相对应的 `control` ：

```ts
control(
    formState: Object,
    validator?: ValidatorFn | ValidatorFn[],
    asyncValidator?: AsyncValidatorFn | AsyncValidatorFn[]
    ): FormControl;
```

此外还值得注意的一点是 address 的处理，我们可以清晰的看到 FormBuilder 支持嵌套，遇到 FormGroup 时仅仅需要再次使用 `this.fb.group({...})` 即可。这样我们的表单在拥有大量的表单项时，构造起来就方便多了。

### 自定义验证

对于响应式表单来说，构造一个自定义验证器是非常简单的，比如我们上面提到过的的验证 `密码` 和 `重复输入密码` 是否相同的需求，我们在响应式表单中来试一下。

```ts
  validateEqual(passwordKey: string, confirmPasswordKey: string): ValidatorFn {
    return (group: FormGroup): {[key: string]: any} => {
      const password = group.controls[passwordKey];
      const confirmPassword = group.controls[confirmPasswordKey];
      if (password.value !== confirmPassword.value) {
        return { validateEqual: true };
      }
      return null;
    }
  }
```

这个函数的逻辑比较简单：我们接受两个字符串（是 FormControl 的名字）,然后返回一个 `ValidatorFn`。但是这个函数里面就奇奇怪怪的，
比如 `(group: FormGroup): {[key: string]: any} => {...}` 是什么意思啊？还有，这个 `ValidatorFn` 是什么鬼？我们来看一下定义：

```ts
export interface ValidatorFn {
    (c: AbstractControl): ValidationErrors | null;
}
```

这样就清楚了， `ValidatorFn` 是一个对象定义，这个对象中有一个方法，此方法接受一个 `AbstractControl` 类型的参数（其实也就是我们的 FormControl，而 AbstractControl 为其父类），而这个方法还要返回 `ValidationErrors` ，这个 `ValidationErrors` 的定义如下：

```ts
export declare type ValidationErrors = {
    [key: string]: any;
};
```

回过头来再看我们的这句 `(group: FormGroup): {[key: string]: any} => {...}`，大家就应该明白为什么这么写了，我们其实就是在返回一个 `ValidatorFn` 类型的对象。只不过我们利用 `javascript/typescript` 对象展开的特性把 `ValidationErrors` 写成了 `{[key: string]: any}` 。

弄清楚这个函数的逻辑后，我们怎么使用呢？非常简单，先看代码：

```ts
    this.user = this.fb.group({
      email: ['', [Validators.required, Validators.email]],
      password: ['', Validators.required],
      repeat: ['', Validators.required],
      address: this.fb.group({
        province: [],
        city: [],
        area: [],
        addr: []
      })
    }, {validator: this.validateEqual('password', 'repeat')});
```

和最初的代码相比，多了一个参数，那就是 `{validator: this.validateEqual('password', 'repeat')}` 。FormBuilder 的 `group` 函数接受两个参数，第一个就是那串长长的，我们叫它 `controlsConfig`，用于表单控件的构造，以及每个表单控件的验证器。但是如果一个验证器是要计算多个 `field` 的话，我们可以把它作为整个 `group` 的验证器。所以 FormBuilder 的 `group` 函数还接收第二个参数，这个参数中可以提供同步验证器或异步验证器。同样还是一个字典对象，是同步验证器的话，`key` 写成 validator，异步的话写成 `asyncValidator` 。

现在我们可以保存代码，启动 `ng serve` 到浏览器中看一下结果了：

![响应式表单对于多值验证的处理](/assets/chap_2_1_011.png)

### FormArray 有什么用？

我们在购物网站经常遇到需要维护多个地址，因为我们有些商品希望送到公司，有些需要送到家里，还有些给父母采购的需要送到父母那里。这就是一个典型的 FormArray 可以派上用场的场景。所有的这些地址的结构都是一样的，有省、市、区县和街道地址，那么对于处理这样的场景，我们来看看在响应式表单中怎么做。

首先，我们需要把 HTML 模板改造一下，现在的地址是多项了，所以我们需要在原来的地址部分外面再套一层，并且声明成 `formArrayName="addrs"`。 FormArray 顾名思义是一个数组，所以我们要对这个控件数组做一个循环，然后让每个数组元素是 FormGroup，只不过这次我们的 `[formGroupName]="i"` 是让 `formGroupName` 等于该数组元素的索引。

```html
<div formArrayName="addrs">
    <button (click)="addAddr()">Add</button>
    <div *ngFor="let item of user.controls['addrs'].controls; let i = index;">
      <div [formGroupName]="i">
        <label>
          <span>省份</span>
          <select formControlName="province">
            <option value="">请选择省份</option>
            <option [value]="province" *ngFor="let province of provinces">{{province}}</option>
          </select>
        </label>
        <label>
          <span>城市</span>
          <select formControlName="city">
            <option value="">请选择城市</option>
            <option [value]="city" *ngFor="let city of (cities$ | async)">{{city}}</option>
          </select>
        </label>
        <label>
          <span>区县</span>
          <select formControlName="area">
            <option value="">请选择区县</option>
            <option [value]="area" *ngFor="let area of (areas$ | async)">{{area}}</option>
          </select>
        </label>
        <label>
          <span>地址</span>
          <input type="text" formControlName="street">
        </label>
      </div>
    </div>
  </div>

```

改造好模板后，我们需要在类文件中也做对应处理，去掉原来的 `address: this.fb.group({...})`，换成 `addrs: this.fb.array([])` 。而

```javascript
this.user = this.fb.group({
  email: ['', [Validators.required, Validators.email]],
  password: ['', Validators.required],
  repeat: ['', Validators.required],
  addrs: this.fb.array([])
}, {validator: this.validateEqual('password', 'repeat')});

```

但这样我们是看不到也增加不了新的地址的，因为我们还没有处理添加的逻辑呢，下面我们就添加一下：其实就是建立一个新的 FormGroup，然后加入 FormArray 数组中。

```ts
  addAddr(): void {
    (<FormArray>this.user.controls['addrs']).push(this.createAddrItem());
  }

  private createAddrItem(): FormGroup {
    return this.fb.group({
      province: [],
      city: [],
      area: [],
      street: []
    })
  }
```

到这里我们的结构就建好了，保存后，到浏览器中去试试添加多个地址吧！

![FormArray 处理结构相同的多组表单项](/assets/chap_2_1_012.png)

### 响应式表单的优势

首先是可测试能力。模板驱动型表单进行单元测试是比较困难的，因为验证逻辑是写在模板中的。但验证器的逻辑单元测试对于响应式表单来说就非常简单了，因为你的验证器无非就是一个函数而已。

当然除了这个优点，我们对表单可以有完全的掌控：从初始化表单控件的值、更新和获取表单值的变化到表单的验证和提交，这一系列的流程都在程序逻辑控制之下。

而且更重要的是，我们可以使用函数响应式编程的风格来处理各种表单操作，因为响应式表单提供了一系列支持 `Observable` 的接口 API 。那么这又能说明什么呢？有什么用呢？

首先是无论表单本身还是控件都可以看成是一系列的基于时间维度的数据流了，这个数据流可以被多个观察者订阅和处理，由于 `valueChanges` 本身是个 `Observable`，所以我们就可以利用 RxJS 提供的丰富的操作符，将一个对数据验证、处理等的完整逻辑清晰的表达出来。当然现在我们不会对 RxJS 做深入的讨论，后面有专门针对 RxJS 进行讲解的章节。

```ts
this.form.valueChanges
        .filter((value) => this.user.valid)
        .subscribe((value) => {
           console.log("现在时刻表单的值为 ",JSON.stringify(value));
        });
```

上面的例子中，我们取得表单值的变化，然后过滤掉表单存在非法值的情况，然后输出表单的值。这只是非常简单的一个 Rx 应用，随着逻辑复杂度的增加，我们后面会见证 Rx 卓越的处理能力。
