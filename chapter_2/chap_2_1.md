# 使用 Angular 快速构造原型

本章会从 Angular 的核心概念出发，第一节以一系列小例子阐释这些概念的意义和使用方法。如果有 Angular 基础的同学可以跳过或者摘选自己感兴趣的内容看。在第二节中，我们会一起来认识 Angular 的官方 UI 组件库 -- Angular Material，这是一套遵循谷歌 Material Design 风格的组件库。使用它的好处在于可以在组件标准化、动画、兼容性方面节省很大精力，即使你不熟悉 CSS 也可以做出很好看的 UI 效果。我们在第二节会一起学习几个较常见的组件。第三节我们会开始我们的一个任务管理系统的原型搭建，当然只是最初的简单框架和页面。后面的章节中，我们会逐渐完善这个例子。最后一节我们会对本章内容进行一个简单回顾和给大家几道小问题进行进一步的思考。

## Angular 基础概念

Angular 中经常提到的几个概念：依赖性注入、模块、组件、指令、管道、模板驱动型表单和响应式表单。这些概念听上去令人头大，加上官网的解释又相对晦涩一些，导致很多人觉得 Angular 的学习门槛好高。但其实这些概念并不复杂，我们就逐一来揭开它们的面纱。

### 依赖性注入

#### 什么是依赖性注入？

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

#### Angular 中的依赖性注入框架

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

#### 依赖性注入进阶

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

