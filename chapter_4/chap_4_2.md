# 响应式编程初探

这里就不谈响应式编程的官方概念了，有兴趣的童鞋可以去 <https://en.wikipedia.org/wiki/Reactive_programming> 去看看。但说老实话，官方概念基本你看完之后还是一头雾水，因为太理论化了。我来谈谈个人的一点粗浅认识，在我看来，响应式编程关注随着时间变化产生的事件或数据，这些事件和数据可以看成随着时间形成的数据流或事件流。估计各位看官一口老血喷出来，你这个解释也是很不着调啊，看不懂，没关系，请记住这个概念，我们随后的学习中你会发现这个概念是很容易理解的。

当然我们在本书中说的响应式编程主要是用的 `Rx` <http://reactivex.io/>, 前端用的是 `rxjs` ，后端是 `RxJava` 。 说到 Rx，这本是微软为自家 .Net 平台打造的，Rx 的本义是 `Reactive Extension` 的缩写，即响应式扩展。距离微软推出 `.Net` 版本的 `Rx` 已经 8 年多了，终于逐渐的获得了主流社区的认可。从单一的 `.Net` 支持到现在的支持 18 种编程语言和 3 个平台框架。那么为什么社区现在对 Rx 的热情这么高呢？Rx 解决了哪些问题？我们接下来就一起来学习。

## 一个不一样的看问题的视角

> “Talk is cheap, show me the code”

废话不说，我们开练了，就先拿上节课没做的“忘记密码”功能当成演示页面。由于要尽快的开始 Rx 的练习，我们就先创建“聪明组件”

```bash
ng g c auth/containers/auth -m auth/auth.module.ts
```

更改模版文件，我们暂时只放一个文本输入框，起一个引用名叫 `myInput`

```html
<input #myInput type="text">
```

在组件文件中，我们改造成下面的样子：

```ts
import { Component, OnInit, ViewChild, ElementRef } from '@angular/core';
import { fromEvent } from 'rxjs';
import { tap, pluck } from 'rxjs/operators';

@Component({
  selector: 'forgot-password',
  templateUrl: './forgot-password.component.html',
  styleUrls: ['./forgot-password.component.scss']
})
export class ForgotPasswordComponent implements OnInit {
  @ViewChild('myInput', { read: ElementRef })
  input: ElementRef;

  ngOnInit(): void {
    const input$ = fromEvent(this.input.nativeElement, 'keyup');
    input$.subscribe(ev => console.log(ev));
  }
}
```

对于上面代码，我们简单做个说明，我们的成员变量 `input` 是通过 `@ViewChild('myInput', { read: ElementRef })` 注解得到了模版中 `myInput` 的引用。

在 `ngOnInit()` 中，我们使用了 `fromEvent` ，这是个 `rxjs` 提供的用于转换事件为 `Observable` 。这个 `Observable` 赋值给一个变量 `input$` ，在 `rxjs` 领域一般在 Observable 类型的变量后面加上 `$` 标识这是一个“流变量”（由英文 `Stream` 得来， `Observable` 就是一个 `Stream` ，所以用 `$` 标识），不是必须的，但是属于约定俗成。

然后我们订阅了这个 `Observable` ，并在 `console` 中输出了订阅的值。

打开 `Chrome` 的开发者工具，在输入框中依次输入 `1` 、  `2` 、 `3` 、 `4` 、 `5` ，同时观察 `console`。

![键盘事件在 console 中的输出](/assets/2018-04-09-23-08-27.png)

我们可以看到伴随着每次键盘的抬起事件， `console` 中也输出了一个 `KeyboardEvent` ，由此可见，我们订阅的是这个文本输入框的 `keyup` 键盘事件 -- `fromEvent(this.input.nativeElement, 'keyup')` 。

如果我们把时间考虑在内的话，上面的这些输出可以看作一个事件流，我们把这个事件流沿时间轴的变化画成下面的示意图，其中 `ke` 代表 `KeyboardEvent` 。我们在不同时间点按了键盘上的数字，从而引发对应的键盘事件，因此我们把它叫做一个事件流。

```txt
      1      2            3     4          5
------ke-----ke-----------ke----ke---------ke----
```

接下来，我们对代码稍做更改，我们即将体验 `rxjs` 中强大的操作符。上面我们取到的是事件对象，但是这个事件对象当中有太多东西是我们不需要的，我们只关心键盘按下的字符。所以我们采用一个操作符 `pluck` ，这个操作符取出对应事件的属性，在这里就是 `ev.target.value` 。 `tap` 也是一个操作符，就是 `rxjs` 5.x 中的 `do` ，在这里我们在值转换之前，使用这个操作法打印事件对象，这个操作符之所以放在 `pluck` 之前是由于执行完 `pluck` 之后流就由事件流转换成了字符流。

```ts
const input$ = fromEvent(this.input.nativeElement, 'keyup').pipe(
      tap(ev => console.log(ev)),
      pluck('target', 'value')
    );
input$.subscribe(val => console.log(val));
```

![增加 pluck 操作符之后的 console 输出](/assets/2018-04-09-23-44-59.png)

`rxjs` 最擅长的就是对流进行各种操作变换，下面的示意图可以让我们了解一个事件流到一个字符流的转换过程

```txt
------ke-----ke-----------ke----ke---------ke----
      |      |            |     |          |
       \      \            \     \          \
       value  value        value value     value
```

## 实现一个计数器

为了让大家有进一步的体会，我们接下来做一个简单的计数器，有 `+1` 和 `-1` 两个按钮，点 `+1` 计数器就加一，点 `-1` 计数器就减一。这个需求很简单吧，我们看看如何使用 `rxjs` 来实现。

![计数器的外观](/assets/2018-04-10-13-56-35.png)

HTML 模版非常简单，一个圆形的显示数字的区域，和两个小按钮。我们为了监听按钮的事件，给两个按钮分别起了一个引用名称 `increment` 和 `decrement` 。

```html
<div class="counter-container">
  <label for="qty">计数器</label>
  <div id="qty" class="numberCircle">{{click$ | async}}</div>
  <div>
    <button class="counter" #decrement>-1</button>
    <button class="counter" #increment>+1</button>
  </div>
</div>
```

接下来看一下组件，对应 HTML 中的两个引用，我们创建了 2 个成员变量 `increment` 和 `decrement` 。并且定义了一个显示计数器数字的流变量 `click$` ，这也意味着这个流是一个数据为数字的流。这个 `async` 是 Angular 为异步变量（比如 `Observable` 或 `Promise` 类型的变量）定制的管道。

```ts
import { Component, ViewChild, ElementRef, OnInit } from '@angular/core';
import { Observable, fromEvent, merge } from 'rxjs';
import { tap, mapTo, scan, startWith } from 'rxjs/operators';

@Component({
  selector: 'app-forgot-password',
  templateUrl: './forgot-password.component.html',
  styleUrls: ['./forgot-password.component.scss']
})
export class ForgotPasswordComponent implements OnInit {
  @ViewChild('increment', { read: ElementRef })
  increment: ElementRef;
  @ViewChild('decrement', { read: ElementRef })
  decrement: ElementRef;
  click$: Observable<number>;

  constructor() {}

  ngOnInit(): void {
    this.click$ = this.getCounterObservable();
  }

  private getCounterObservable(): Observable<number> {
    const increment$ = fromEvent(this.increment.nativeElement, 'click').pipe(
      tap(_ => console.log('increment')),
      mapTo(1)
    );
    const decrement$ = fromEvent(this.decrement.nativeElement, 'click').pipe(
      tap(_ => console.log('decrement')),
      mapTo(-1)
    );
    return merge(increment$, decrement$).pipe(
      scan((acc: number, curr: number) => acc + curr),
      startWith(0)
    );
  }
}
```

我们在 `ngOnInit` 时给 `click$` 赋值，请注意为了更好的隔离，我们抽取出一个专门的函数来做流处理 `getCounterObservable` 。

下面我们看一下我们是如何处理这个流的，首先来分析一下原始的场景：页面上有两个按钮，点击 `+1` 会产生一个点击事件，点击 `-1` 同样也会产生事件。也就是说页面中会产生 2 个独立事件流，如下所示：

```txt
+1按钮：---i----i-------i------i-------i-----i----
-1按钮：------d-----d-------------d-------d-------
```

但这两个事件导致的结果是有区别的，前者会导致数据 +1 ，而后者会导致数据 -1 。如果我们想体现这个结果的话，最方便的做法就是在 `+1` 按钮点击事件产生时，将其转换成数字 `1` ，而在 `-1` 按钮点击事件产生时，将其转换成 `-1` ，应用 `mapTo` 操作符，就可以达到这个效果。

```txt
+1按钮：---i----i-------i------i-------i-----i----
          \     \       \      \       \     \
           1     1       1      1       1     1
-1按钮：------d-----d-------------d-------d-------
             \     \              \      \
             -1    -1             -1     -1
```

但现在我们面临一个问题，最终在页面上我们只有一个数字流，但现在却有两个，这个局怎么破？之前我们说过 Rx 最擅长的就是对流进行各种操作，所以这个问题对于 Rx 来说再简单不过，这两个流需要一个合并操作符。Rx 提供了很多合并操作符，具体用什么操作符需要根据合并的规则，这个例子里面，我们的这两个数据流其实做一个简单合并就可以，简单合并就是保持各自流中的顺序和时间点，可以想象成一个流向另一个做投影，从而产生了一个新的流。

```txt
+1 按钮：---i----i-------i------i-------i-----i----
-1 按钮：------d-----d-------------d-------d-------
合并流： ---i--d-i---d---i------i--d----i--d---i---
           \  \  \  \    \     \   \   \   \   \
           1  -1  1  -1   1     1   -1  1  -1   1
```

但接下来又出现了一个问题，我们得到了一个数字流，这个数字流由 1 和 -1 构成，但我们需要的是一个累加值，这个累加值才是应该显示在计数器伤的数字。所以就又引入了一个新的操作符 `scan` ，它允许你使用一个累加器函数去操作这个流中的数据，并返回即时的结果。也就是说，原始流中的每一个数据都会产生对应的累加器返回的结果。我们使用的累加器函数非常简单 `(acc: number, curr: number) => acc + curr` 就是返回 `累加器 (acc) + 当前数据 (curr)` 的和，而等到下一个数据出现时，累加器的值就是上一次计算后的结果。那么第一个数据产生时，没有累加器的值怎么办呢，没关系，默认值是 0 ，正好符合我们的需求。

```txt
+1 按钮：---i----i-------i------i-------i-----i----
-1 按钮：------d-----d-------------d-------d-------
合并流： ---i--d-i---d---i------i--d----i--d---i---
           \  \  \  \    \     \   \   \   \   \
           1  -1  1  -1   1     1   -1  1  -1   1
累加器： ---1---0--1---0---1-----2----1--2---1---2
```

最后有一个问题是如果页面加载后，我们没有点击任何按钮时，这个计数器是没有值的，因为没有事件产生，所以我们要给个初始值 0 ，这个就是操作符 `startWith` 的意义。

当然前面没有给出样式，这里给出一个样式文件供大家参考，但这个样式和功能是没有什么关系的。

```css
:host {
  display: flex;
  flex-direction: column;
  justify-content: center;
  flex: 1 1 auto;
}
.numberCircle {
  border-radius: 50%;
  flex: 1;
  width: 36px;
  height: 36px;
  padding: 8px;
  background: #fff;
  border: 2px solid #666;
  color: #666;
  text-align: center;
  font: 32px Arial, sans-serif;
}
.count-down {
  color: #333;
  text-align: center;
  background-color: #ffd54f;
}
h1 {
  font-weight: normal;
}
li {
  display: inline-block;
  font-size: 1.5em;
  list-style-type: none;
  padding: 1em;
  text-transform: uppercase;
}
li span {
  display: block;
  font-size: 4.5rem;
}
.counter-container {
  /* basics */
  background-color: #444;
  color: #c4be92;
  text-align: center;
  display: flex;
  flex-direction: column;
  align-content: space-around;
  align-items: center;

  /* rounded corners */
  -webkit-border-radius: 12px;
  border-radius: 12px;
  -moz-background-clip: padding;
  -webkit-background-clip: padding-box;
  background-clip: padding-box;
  padding: 0.8em 0.8em 1em;
  width: 8em;
  margin: 0 auto;
  -webkit-box-shadow: 0px 0px 12px 0px #000;
  box-shadow: 0px 0px 12px 0px #000;
}
button.counter {
  /* basics */
  color: #444;
  background-color: #b5b198;
  /* rounded corners */
  -webkit-border-radius: 6px;
  border-radius: 6px;
  -moz-background-clip: padding;
  -webkit-background-clip: padding-box;
  background-clip: padding-box;
  font-weight: bold;
  width: 30px;
}
button.counter:hover,
button.counter:active,
button.counter:focus {
  background-color: #cbc7ae;
}
```

### 和传统方式的比较

你可能会说，这也没什么啊，感觉比传统的事件处理也高明不到哪里去，好像还更繁琐一些。那么好我们看看传统方式，我们怎么写这段程序。

首先是定义一个成员变量 `counterNum` ，然后给出 +1 对应的处理函数 `processIncrement` 和 -1 对应的处理函数 `processDecrement` 。这两个函数中分别对成员变量 `counterNum` 进行 `+1` 或 `-1` 的操作就好。

```ts
import { Component, OnInit } from '@angular/core';

@Component({
  selector: 'app-forgot-password',
  templateUrl: './forgot-password.component.html',
  styleUrls: ['./forgot-password.component.scss']
})
export class ForgotPasswordComponent implements OnInit {
  counterNum = 0;

  constructor() {}

  ngOnInit(): void {
  }

  processIncrement() {
    ++this.counterNum;
  }

  processDecrement() {
    --this.counterNum;
  }
}
```

然后在模版中进行事件绑定，一切就 ok 了。

```html
<div class="counter-container">
  <label for="qty">计数器</label>
  <div id="qty" class="numberCircle">{{ counterNum }}</div>
  <div>
    <button class="counter" (click)="processDecrement()">-1</button>
    <button class="counter" (click)="processIncrement()">+1</button>
  </div>
</div>
```

真的是很简单啊，不过且慢，让我们加一个小需求试试看。现在我们希望鼠标一直按着 `+1` 或 `-1` 时，数字是会自动连续增加或减少的，而且频率是 `300ms` 更新一个数字。在鼠标抬起时，这个数字就要停止更新了。

使用传统模式的话，我们看看需要怎么做。首先需要再引入一个布尔型的成员变量 `mouseUp` ，用这个变量标识是否鼠标已抬起。然后在 `processIncrement` 和 `processDecrement` 中加入一个 `setInterval` 计时器对 `counterNum` 进行加或减，而且需要判断 `mouseUp` 为真时取消计时器。在这两个方法中需要将 `mouseUp` 置为 `false` 。还需要增加一个鼠标抬起的事件处理函数 `processMouseUp` ，在其中设置 `mouseUp` 为 `true` 。

```ts
// 省略其他部分
export class ForgotPasswordComponent implements OnInit {
  counterNum = 0;
  mouseUp = false;

  constructor() {}

  ngOnInit(): void {
  }

  processIncrement() {
    this.mouseUp = false;
    const that = this;
    const timer = setInterval(function() {
      ++that.counterNum;
      if (that.mouseUp) {
        clearInterval(timer);
      }
    }, 300);
  }

  processDecrement() {
    this.mouseUp = false;
    const that = this;
    const timer = setInterval(function() {
      --that.counterNum;
      if (that.mouseUp) {
        clearInterval(timer);
      }
    }, 300);
  }

  processMouseUp() {
    this.mouseUp = true;
  }
}
```

同样的在模版中，需要绑定两个事件 `mousedown` 和 `mouseup`

```html
<div class="counter-container">
  <label for="qty">计数器</label>
  <div id="qty" class="numberCircle">{{counterNum}}</div>
  <div>
    <button class="counter" (mousedown)="processDecrement()" (mouseup)="processMouseUp()">-1</button>
    <button class="counter" (mousedown)="processIncrement()" (mouseup)="processMouseUp()">+1</button>
  </div>
</div>
```

怎么样是不是越来越复杂了，而且引入的标志位多了起来，处理的函数也越来越多，如果我们又有新的需求，就会导致逻辑越来越读懂，可维护性也越来越差了。

那么我们看看使用 Rx 怎么解决这个问题

### 简洁而强大的 Rx 解决方案

和最初的 Rx 方案对比，我们其他的文件都不用变化，只是调整一下 `src/app/auth/containers/forgot-password/forgot-password.component.ts` 的 `getCounterObservable` 方法：

```ts
private getCounterObservable(): Observable<number> {
  const mouseUp$ = fromEvent(document, 'mouseup');
  const increment$ = fromEvent(
    this.increment.nativeElement,
    'mousedown'
  ).pipe(
    switchMap(_ =>
      interval(300).pipe(startWith(1), takeUntil(mouseUp$), mapTo(1))
    )
  );
  const decrement$ = fromEvent(
    this.decrement.nativeElement,
    'mousedown'
  ).pipe(
    switchMap(_ =>
      interval(300).pipe(startWith(1), takeUntil(mouseUp$), mapTo(-1))
    )
  );
  return merge(increment$, decrement$).pipe(
    scan((acc, curr) => acc + curr),
    startWith(0)
  );
}
```

我们的思考方式和原来的类似，只不过现在页面中有三个事件流，两个按钮的鼠标按下产生的事件流（从前面的 `click` 事件改成了 `mousedown` 事件）以及鼠标抬起的事件流 `mouseup`。最初我们是直接把两个按钮的事件转换成了数字，现在由于需求的变化，我们不能直接转换。

先来分析一下，当鼠标按下时，我们希望产生一个计时器，而这个在 Rx 内建提供了 `interval` ，这个计时器每 `300ms` 产生一个顺序整数，但是第一个数是 `300ms` 后才发射的，所以我们使用了 `startWith(1)` 让它一开始就发射一个数据。这么做的原因是我们希望点击按钮（而不是长按）时可以立即变更该数字。

再有就是这个计时器应该在 `mouseup` 事件产生时停止，所以我们使用 `takeUntil` 操作符，这个操作符的意义在于当其参数的流有数据产生时，外部的流就完成了，也就停止了。比如 `interval(300).pipe(startWith(1), takeUntil(mouseUp$))` ，当 `mouseUp$` 有数据时，也就是事件触发时，那么 `interval(300)` 就停止了。

然后我们还是把流的数据 `mapTo` 成 `1` 或 `-1` ，但为什么外层要用 `switchMap` 呢？这是因为内部的 `interval.pipe(...) ` 得到的是一个 `Observable<number>` 而不是 `number` ，这就是数据流中又有数据流了，所以使用一个高阶的操作符 `switchMap` 将它“拍扁”，重新变成 `number` 型。这一块会在后面讲到 Rx 高阶操作符时详细阐述。

总之，你可以看到使用 Rx ，可以让我们的逻辑更加简洁、清晰的表达出来，而且不需要引入一些标志位。

## 为什么要使用 Rx？

个人以为 Rx 的两大优点：

1. 由于在 Rx 世界里，一切都是事件流，所以这『逼迫』开发者将时间维度纳入设计的考量，从而需要分析清楚每个过程的，从开始到结束，确保过程可控制。
2. 提供的各种强大的操作符可以将逻辑非常轻松的组合，减少分散在代码各个角落的逻辑，使得逻辑的可读性更强。

### rxjs 6.x 和之前版本的区别

本书中的代码是基于 `rxjs 6.x` 的，和 `5.x` 有较大区别，从 6.x 开始起，不再提倡原来的类似 `some$.map(x => x/2).filter(x => x > 1)` 这样的链式写法。原因是这种方式容易污染 `Observable` 原型，而且在 Angular 无法通过摇树去删除无用代码。所以现在都推荐使用 `pipeable` 操作符。我们看到前面例子中的 `pipe` 就是接受一系列的 `pipeable` 操作符，在这个 pipe 方法中，放入一系列操作符，基本的思路和之前是一致的，只不过写法上比链式写法要稍微繁琐一些。

## Observable 的性质

Rx 的本质是观察者模式，所以我们需要知道 `Observable` 就是一个可观察对象，与之对应的有 `Observer` -- 观察者和 `Subscriber` 订阅者。

`Observer` 是一个 `Observable` 所产生的推送消息的的消费者。

```ts
interface Observer<T> {
  closed?: boolean;
  next: (value: T) => void;
  error: (err: any) => void;
  complete: () => void;
}
```

从上面的定义可以看到 `Observer` 是一个接口，如果我们有一个具体 `observerA` 实现了该接口，有一个可观察对象 `ob$`，那么 `ob$.subscribe(observerA)` 就是让 `observerA` 订阅了 `ob$` 。当 `ob$` 有新的数据或事件产生时，它会调用 `observerA` 的 `next(value)` 方法 （这个 `next` 就是在 `Observer` 接口中定义的），从而让 `observerA` 不断的接收到 `ob$` 的变化。

我们在上面的例子中没有做 `subscribe` ，这是由于我们使用了 Angular 提供的 `async` 管道。其实我们也可以采用 `subscribe` 。我们把 `click$: Observable<number>` 改成 `click: number;` ，当然模版的绑定也改成 `{{click}}` ，然后把 `ngOnInit` 改写成下面的样子。

```ts
ngOnInit(): void {
  this.getCounterObservable().subscribe({
    next: val => {
      this.click = val;
    }
  });
}
```

运行程序，你会发现效果是一样的。现在我们来分析一下， `this.getCounterObservable()` 得到的是一个 `Observable` ，那么我们使用了 `subscribe` 方法，这个方法接收的是一个 `Observer` 类型的参数。我们也构造了一个对象，尽管只有一个 `next` 属性。

```ts
{
  next: val => {
    this.click = val;
  }
}
```

这么写还是有点麻烦对吗？好在我们还有一些语法糖可用，上述形式可以简写为

```ts
this.getCounterObservable().subscribe(val => {
      this.click = val;
    });
```

而且在只有一行代码时可以进一步简写

```ts
this.getCounterObservable().subscribe(val => this.click = val);
```

这样看起来好多了，对吧。那么其他几个参数呢？我们可以给出三个参数，也就是说 subscribe 方法提供了一些语法糖的形式，可以不用完整的传入 Observer 对象，而是以参数形式提供 `subscribe(nextFn, errorFn, completeFn)` ，这三个参数都是函数形式，而且不用全部提供。

```ts
this.getCounterObservable().subscribe(
  val => this.click = val,
  err => console.error(err),
  () => console.log('completed'));
```

这三个参数分别是 `Observable` 接口中定义的 `next` 、 `error` 和 `complete` ，都是函数型参数。这个也可以解释任何一个 `Observable` 都有三个状态：等待推送下一个值就是 `next` ，出错了就是 `error` ，流结束了就是 `complete` 。无论是正常结束还是出了 `error` 结束都会调用 `complete`。 比如说我们有一个 `from([1,2,3])` ，这个流有三个数据，这三个发送完了，就没有 next 了，也就调用 `complete` 了，这个就是正常结束。再比如 `from([1,2,3]).pipe(map(val => val/val-2))` 在第二个数据时由于被除数为 0 ，会调用 `error` ，然后走 `complete` 。

但是这样直接 subscribe 会有内存泄漏问题，因为这个订阅一直在那里，我们怎么取消订阅呢？这就要请出 `Subscription` 了。 `subscribe` 这个方法返回的就是一个订阅对象 `Subscription` ，那一般情况下我们使用一个成员变量存储，然后在组件销毁时 `ngOnDestroy` 使用 `unsubscribe` 方法取消订阅。这样才能保证内存不会泄漏，但这样做总会由于比较麻烦导致有些时候忘记这么做，所以我们在 Angular 中推荐尽可能的使用 `async` 管道。

```ts
import { Observable, fromEvent, merge, interval, Subscription } from 'rxjs';
// 省略其他部分
sub: Subscription

ngOnInit() {
  this.sub = this.getCounterObservable().subscribe(
    val => this.click = val,
    err => console.error(err),
    () => console.log('completed'));
}

ngOnDestroy {
  if (this.sub) {
    this.sub.unsubscribe();
  }
}
```
