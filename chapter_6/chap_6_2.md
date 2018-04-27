# rxjs 进阶

## 改造登录表单

我们这里给登录表单增加一个图形验证码功能，当然后端已经集成了这个功能。这个功能对于笨组件来说非常简单，模版上增加一个 `<input>` 和 `<img>` 控件，注意这个控件此时并没有作为表单控件出现（没有设置 formControlName），因为现在我们的鉴权接口中没有图形验证码的参数。那可能有的同学会问了，这个验证如果和鉴权无关的话，那么不是可以直接绕过去了吗，比如使用 Postman 直接访问鉴权接口，就不用验证码了啊。对的，这个地方是有问题的，但是我们放到后面处理，现在先实现验证码的显示和检验。

```html
<mat-card>
  <!--省略头部-->
  <mat-card-content>
    <form fxLayout="column" fxLayoutAlign="stretch center" [formGroup]="form" (ngSubmit)="submit(form, $event)">
      <!--省略其他 field -->
      <mat-form-field class="full-width">
        <input matInput placeholder="图形验证码" (input)="verifyCaptcha($event.target.value)">
        <img matSuffix [src]="captchaUrl" (click)="processClick()">
      </mat-form-field>
      <!--省略按钮 -->
    </form>
  </mat-card-content>
  <!--省略 Action -->
</mat-card>
```

在组件文件中我们简单的把图片点击事件和输入框的输入事件发射出去，并提供一个图片地址的输入型属性。

```ts
// 省略
export class LoginFormComponent implements OnInit {
  // 省略
  @Input() captchaUrl = '';
  @Output() refreshCaptcha = new EventEmitter<void>();
  @Output() codeInput = new EventEmitter<string>();
  // 省略
  processClick() {
    this.refreshCaptcha.emit();
  }
  verifyCaptcha(code: string) {
    this.codeInput.emit(code);
  }
}
```

在“聪明组件”（ `LoginComponent` ）中，我们的模版文件要设置上笨组件的输入和输出属性。

```html
<div fxLayout="row" fxLayout.xs="column" fxLayoutAlign="center space-around">
  <app-login-form
    (submitEvent)="processLogin($event)"
    (refreshCaptcha)="refreshCaptcha()"
    (codeInput)="verifyCaptcha($event)"
    [captchaUrl]="(captcha$ | async)?.captcha_url">
  </app-login-form>
</div>
```

## rxjs 的高阶操作符

`LoginComponent` 中涉及到图片验证码的逻辑中有以下几个流：

* 点击图片的点击事件流 `clickSub`
* 输入框的输入事件流 `verifySub`
* 请求验证码图片的 API 事件流 `this.authService.requestCaptcha()`
* 验证输入的验证码的 API 事件流 `this.authService.verifyCaptcha(token, code)`

但这几个流需要并不是孤立存在的，而是有一定的关联关系

1. `clickSub` 触发之后才引发 `this.authService.requestCaptcha()`
2. `verifySub` 触发之后才引发 `this.authService.verifyCaptcha(token, code)`

这里面体现了两层意思，第一个是时间顺序，第二个是 **流中嵌套了流** 。先来看组件代码

```ts
// 省略元数据和导入声明
export class LoginComponent implements OnDestroy, OnInit {
  // 省略 quote$ 声明
  captcha$: Observable<Captcha>;
  clickSub = new Subject();
  captchaSub = new BehaviorSubject<Captcha>(null);
  verifySub = new Subject<string>();
  sub = new Subscription();
  // 省略构造函数
  ngOnInit(): void {
    this.captcha$ = this.clickSub.pipe(
      startWith({}),
      switchMap(_ => this.authService.requestCaptcha()),
      tap(captcha => {
        this.captchaSub.next(captcha);
      })
    );

    this.sub = this.verifySub
      .pipe(
        withLatestFrom(this.captchaSub),
        switchMap(([code, captcha]) =>
          this.authService
            .verifyCaptcha(captcha.captcha_token, code)
            .pipe(
              map(res => res.validate_token),
              catchError(err => of(err.error.title))
            )
        )
      )
      .subscribe(t => console.log(t));
  }
  ngOnDestroy() {
    if (this.sub) {
      this.sub.unsubscribe();
    }
  }
  processLogin(auth: Auth) {
    this.authService
      .login(auth)
      .pipe(take(1))
      .subscribe(u => console.log(u));
  }
  refreshCaptcha() {
    this.clickSub.next();
  }
  verifyCaptcha(code: string) {
    this.verifySub.next(code);
  }
}
```

在上面代码中 `switchMap` 是一个高阶操作符，很多同学看到“高阶”这个词就有点懵逼，简单类比一下， `x+2=4` 这是 `x` 的一次方程，那么 `x*x+2=4` 这就是 `x` 的二次方程，也可以说是 `x` 的高阶方程。所以类似的， `Observable<Type>` 这是“一次”的 `Observable` ，那么 `Observable<Observable<Type>>` 就是高阶的 `Observable` 。

拿上面的例子说话， `this.clickSub` 得到的是 `void` 的数据流，因为我们 `next()` 时没有传递数据，这会发送一个 `void` ，也就是我们只需知道事件发生了，不需要关心具体的值。如果我们不采用 `switchMap` 的话，那么可以这样写 `this.clickSub.pipe(map(_ => this.authService.requestCaptcha()))` ，但是这样转换之后我们得到流中的值就由 `void` 变成了 `Observable<Captcha>` ，看，我们在流中又发现了流。那么设想一下，如果这个时候我们订阅这个流

```ts
this.clickSub.pipe(map(_ => this.authService.requestCaptcha()))
.subscribe(val => console.log(val))
```

大家可以自行实验一下，输出的应该是一个 `Observable` 对象。但如果我们想要得到的是那个 `Captcha` 怎么办呢？由于 `val` 是 `Observable<Captcha>` ，所以我们就再订阅一下

```ts
this.clickSub.pipe(map(_ => this.authService.requestCaptcha()))
.subscribe(val => {
  val.subscribe(captcha => console.log(captcha));
})
```

大家再实验一下，ok，问题解决了，但，且慢。总觉得有点不对劲呢，这个解决方案看起来总是有点别扭。

首先，需要知道一点， `subscribe` 意味着流的终结。 `subscribe` 意味着流的终结。 `subscribe` 意味着流的终结。重要的事说三遍，一旦 `subscribe` 之后，你就无法再利用各种操作符了。

```ts
his.clickSub.pipe(map(_ => this.authService.requestCaptcha()))
.subscribe(val => {
  val.subscribe(captcha => console.log(captcha))
  .filter(captcha => captcha !== null); // <--- 不正确，无法再应用操作符
})
```

其次，如果采用上面的方式，你将无法在希望的时候取消内部的订阅，异常处理也是非常麻烦。

最后，你将陷入一个嵌套地狱，这正是 rxjs 要解决的问题，但你这么使用其实就又回到了老路上。

请记住以下几个原则

* 永远不要在 `subscribe` 中嵌套 `subscribe` ，高阶操作符就是处理这类需求的。
* 尽可能少的 `subscribe` ，尽量合并流，或者利用 `async` 管道
* 如果 `subscribe` ，一定记得要在适当的时候取消订阅 `unsubscribe` ，否则会内存泄露

高阶操作符的作用就是将高阶转为低阶，起到流的拍扁的作用，很多同学初学时不知道什么时候该使用高阶操作符，一个技巧可以先写成低阶，然后看操作符内返回的是流还是你要的数据类型，如果是流就使用高阶。

### mergeAll/mergeMap/switchMap

#### mergeAll

还记得我们前面提到的嵌套 subscribe 的例子？我们首先使用普通的 `map` 操作符得到一个高阶流

```ts
// 高阶流
this.captcha$ = this.clickSub.pipe(
  map(_ => this.authService.requestCaptcha()),
);
// mergeAll 拍扁这个高阶流
this.captcha$.mergeAll().subscribe(captcha => console.log(captcha));
```

`mergeAll` 其实做的事情是订阅了内层流（ `this.authService.requestCaptcha()` ）然后把内层的值发送到外层。

#### mergeMap

那么这种操作在 rxjs 中太频繁了，所以有了一个快捷方式 -- `mergeMap` ，你不用分两步处理了，一步到位。

```ts
this.captcha$ = this.clickSub.pipe(
  mergeMap(_ => this.authService.requestCaptcha()),
);
```

所以其实 `mergeMap` 相当于 `map` + `mergeAll`

但这种方式有个问题，就是对于每个外层 `Observable` 的值都会产生内层流，用我们具体的例子来看，就是每次点击图片（外层流）都会去发送 API 请求（内层流）。这个如果是你想要的，那么就用 `mergeMap` 。

但通常情况下，如果用户快速点击图片的话，会产生多个 HTTP 的 API 请求，但我们其实只关心最新的这次，之前的我们不需要。这就需要请出 `switchMap` 了。

#### switchMap

那么我们用的这个 `switchMap` 究竟起到什么什么作用呢？首先它为什么有个 `switch` 的前缀，就是因为要切换的意思，它代表着，如果外层 `Observable` 有值发出，就立刻取消之前的订阅，重新开始一个新的订阅。我们只维护一个内层流的订阅，就是最新的这个外层 `Observable` 对应的内层流。

这样解释还是太抽象了，我们来看上面的例子中的一段代码，如果 `this.clickSub` 有新值，也就是用户又点了一下，那么我们就取消之前的内层流订阅，只关心这次的点击产生的内层流 `this.authService.requestCaptcha()` 。

```ts
this.captcha$ = this.clickSub.pipe(
  // 忽略其他操作符
  switchMap(_ => this.authService.requestCaptcha()),
  // 忽略其他操作符
);
```

### Subject 以及事件流的“冷”和“热”

重点看一下组件的改造，我们之前都是 `from` 或 `fromEvent` 由现存对象或事件构建一个流，现在让我们尝试一个新的方式，使用 `Subject` 。

我们之前讲过 `Subject` 是一个奇葩的存在，它既是 `Observer` 又是 `Observable` 。它是 `Observer` ，所以有 `next(v)` 、 `error(e)` 和 `complete()` 。它又是 `Observable` ，所以适用于 `Observable` 的操作符都适用于它，它也可以 `subscribe()` 。

另一个明显的区别是， `Subject` 构成的流和普通 `Observable` 不一样，怎么个不一样法呢？看下面的例子的代码，我们声明了 `clickSub` ，在图片点击事件的处理函数 `refreshCaptcha` 中，发射了一个事件 `this.clickSub.next()` 。那么无论我们订阅与否，这个流中都有数据，对吗？因为事件触发了，这个流就有值发射出来。但如果我们使用类似 `fromEvent(imageElement, 'click')` 这种形式构建 `Observable` 的话，这个 `Observable` 在没有任何订阅的时候是 **没有值** 的。

这就引出了一个流的“冷”和“热”的概念，类似 `Subject` 这种流就是“热”的流，而普通的 `Observable` 是“冷”的流。举一个更形象的例子来说明“冷”和“热”的区别：冷的流就好像我们去视频网站看电影，你和我分别在两台电脑上点进去，我是下午 2:00 进去看的，你是下午 2:30 进去看的，但我们都会从头看起，而不是你点进来一看已经进行了半个小时了。而热的流就像我们分别在各自家中看球赛的直播，那么我是 2:00 打开电视的，你是 2:30 打开的，但你就只能错过前半个小时了，我们看到的内容是一样的。

再举一个小例子，下面的两个订阅，你认为应该分别打印出什么呢？对的，都是 `1,2,3,4`

```ts
const data$ = Observable.from([1,2,3,4]);
const subscriptionA = data$.subscribe(d => console.log(d));
const subscriptionB = data$.subscribe(d => console.log(d));
```

那么如果是下面的代码呢？ `subscriptionA` 和 `subscriptionB` 这两个订阅会打印什么？是的，`subscriptionA` 的输出是 `1,2,3,4` 而 `subscriptionB` 的输出是 `3, 4` ，`subscriptionB` 由于订阅的时间点在 `data$.next(2);` 之后，所以完美的错过了前两个值。

```ts
const data$ = new Subject<number>();
const subscriptionA = data$.subscribe(d => console.log(d));
data$.next(1);
data$.next(2);
const subscriptionB = data$.subscribe(d => console.log(d));
data$.next(3);
data$.next(4);
```

那么好，究竟是使用冷的流还是热的呢？答案是依据你的需求和场景，各有各的用途。当然很多时候，其实它们可能都可以解决你的问题，但要注意它们的区别，在遇到结果和你的期望不符时，请仔细想想你是否忽略了流的性质。

#### 可以记住上一值的 Subject -- BehaviorSubject

在 `LoginComponent` 中，我们还使用了一个不同的 Subject -- BehaviorSubject （ `captchaSub = new BehaviorSubject<Captcha>(null);`） ，它的不同之处在于它可以记住最近的一个值，这个太有用了。我们来看一下 captcha 获取和验证的逻辑

1. 发送 GET 请求到 `/api/auth/captcha`
2. 后端 API 返回 `{ captcha_token: 'blabalbla', captcha_url: 'http://xxx.yyy.zzz/someimage.jpg' }`
3. 用户输入图片中的代码
4. 发送 POST 请求到 `/api/auth/captcha` ，但需要携带一个 `json` 数据 `{ captcha_token: 'blabalbla', captcha_code: '用户输入值'}`

如果仔细看一下第 4 步，我们需要得到在第 2 步中得到的服务器返回值，也就是 `this.authService.requestCaptcha()` 这个流的发送的最近的值。但是由于这个流是一个冷的流，如果我们使用 `withLatestFrom` 操作符，这个流会又执行一遍，我们得到的就不是上次的值，而是变成了一个新的图片的 token ，这可就麻烦了。

但是没关系，我们请出 `BehaviorSubject` 帮我们搞定。我们在得到验证码的之后将其值用 `BehaviorSubject` 发出（ `this.captchaSub.next(captcha)` ），然后在合并

```ts
this.captcha$ = this.clickSub.pipe(
  startWith({}),
  switchMap(_ => this.authService.requestCaptcha()),
  tap(captcha => {
    this.captchaSub.next(captcha); // <--- 这里发射
  })
);
this.sub = this.verifySub.pipe(
  withLatestFrom(this.captchaSub), // <--- 这里使用最近的值
  switchMap(([code, captcha]) =>
    this.authService
      .verifyCaptcha(captcha.captcha_token, code)
      .pipe(
        map(res => res.validate_token),
        catchError(err => of(err.error.title))
      )
  )
)
.subscribe(t => console.log(t));
```

## 合并操作符

我们之前或多或少的其实已经用上了合并类操作符了，这里我们就做一个集中的介绍。当然 rxjs 提供操作符太多了，我们不会逐一介绍，而是把最常用的几种做一个讨论。

### merge

`merge` 是最简单的一种合并，就是把两个流按各自的顺序合并成一个。

```ts
const dataA$ = interval(1000).pipe(take(5), mapTo('A'));
const dataB$ = interval(2000).pipe(take(5), mapTo('B'));
const merge$ = merge(dataA$, dataB$);
merge$.subscribe(val => console.log(val));
```

输出结果应该是，注意 `AB` 不是一起输出的，而是时间间隔非常小，几乎同时，但流是不可能一下给出两个值的，所以有先后顺序，`A` 在 `B` 前仅仅作为示意，实际上有可能 `B` 在 `A` 前。

```txt
源流A：---A---A---A---A---A
源流B：-------B-------B-------B-------B-------B
合并流:----A--AB--A---AB--A---B-------B-------B
```

### concat 和 startWith

concat 是有顺序的把一个流排在另一个流的末尾，也就是严格按顺序输出，即使排在后面的流实际的速度较快也要等到前面的流完成后才能开始。这和 merge 是不同， merge 是不保证顺序的。

```ts
const dataA$ = of(4, 5, 6);
const dataB$ = of(1, 2, 3);
const concat$ = dataA$.pipe(concat(dataB$));
concat$.subscribe(val => console.log(val));
```

上面的代码输出的结果是 `4,5,6,1,2,3` ，对于 `concat` 大家一定要注意的是不要把 `Observable` 放在一个无尽流之后。什么是无尽流？就是永远不结束，比如 `fromEvent` 就是无尽流，比如 `interval(1000)` 就是无尽流。如果把一个流 `concat` 到一个无尽流后就会导致根本等不到那个流的数据了。

`startWith` 的作用和 `concat` 相反，是在流发射前加一个值，也就是先发射一个初始值。这个操作符很有用，因为有时候，我们构造了一个流，但在没有事件触发的时候，我们也希望有初始值。或者需要一个初始值才能激活这个流。

### combineLatest/zip/withLatestFrom

为什么这几个操作符要放在一起讲？因为它们很相似，但又有差别，这个差别有时很微妙，所以需要放在一起比较着讲。

#### combineLatest

`combineLatest` 的作用是，只要参与合并的任何一个流发射一个值，就把所有参与合并的流的最新值都发出来。但是需要注意的一点是这个合并流是要等参与合并的每个流都发射过值之后才会有值。

这个操作符的最佳使用场景是几个流彼此依赖去进行某种计算或操作，也就是说每个流的变化都会影响计算结果，比如下面的 BMI 计算器中 BMI 的值是由身高和体重两个值计算而得，任何一个值的改变都会影响计算结果。

```ts
const weight$ = fromEvent(weight, 'input').pluck('target', 'value');
const height$ = fromEvent(height, 'input')
  .pluck('target', 'value');

const bmi$ = combineLatest(weight$, height$, (w, h) => {
  return w/(h/100*h/100);
});

bmi$.subscribe(val => {
  bmi.innerText = Math.round(val * 100) / 100;
});
```

![BMI 计算器](/assets/2018-04-25-20-30-26.png)

我们来简单模拟一下各个流的情况：

* 一开始，输入身高 170 ，BMI 值不会显示，因为此时体重数据流还没有值发出。记住 combineLatest 需要每个参与流都至少有一个值发射出来才会有值。
* 然后输入体重 70 ，这时身高流的最新值是 170 ，所以 （170，70）作为合并后的数据参与计算。
* 我们又把身高改成了 180 ，这时体重的最新值是 70 ，所以（180，70）参与计算
* 体重被改成 75 后，身高的最新值未 180 ，那么 （180，75）参与计算。

```txt
身高：----170-----------180------------
体重：---------70------------------75--
BMI：------(170,70)---(180,70)---(180,75)
```

#### zip

`zip` 和 `combineLatest` 非常像，但它要求任何一个参与合并的流发出数据后，等待其他流发出和它位置一致的数据。换句话说，如果流 A 和 B 参与合并，那么 A 流发出第一个元素 A1，合并流会等待 B 发出第一个元素 B1，如果发出，合并流就会有（A1，B1）；如果 B 发出第二个元素 B2 ，合并流会等待 A 发出第一个元素 A2，如果发出，合并流就会有（A2，B2）。

```ts
const dataA$ = interval(1000);
const dataB$ = interval(1000).pipe(take(2));
const zip$ = zip(dataA$, );
//输出: [0,0]...[1,1]
const subscribe = zip$.subscribe(val => console.log(val));
```

上面代码的输出只有 `[0,0]` 和 `[1,1]` ，尽管 `dataA$` 是一个无尽序列，一直有值输出，但合并流并不会，因为它始终等不到 `dataB$` 的第三个元素。

大家如果了解 zip 这个单词有拉链的意思，就会有助于你记住这个操作符的含义，一个齿对应一个齿，有对齐的效果。而且 zip 会等待合并流中最慢的那个，所以有时也会起到延时的作用。

#### withLatestFrom

`withLatestFrom` 就是要在一个流发出最新值时，得到另一个流的最新值。这个听上去也好像 `combineLatest` 啊，但是 `withLatestFrom` 有“以我为主”的意思。举个例子，只要输入框的输入事件发生了，那么此时我去合并 `this.captchaSub` ，但如果 `this.captchaSub` 发射了新值，我们并不在意。

```ts
this.verifySub.pipe(
  withLatestFrom(this.captchaSub), // <--- 这里使用最近的值
  switchMap(([code, captcha]) =>
    this.authService
      .verifyCaptcha(captcha.captcha_token, code)
      .pipe(
        map(res => res.validate_token),
        catchError(err => of(err.error.title))
      )
  )
)
```
