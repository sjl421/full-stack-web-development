# 前端服务层

## 构建“伪”服务

为什么叫做“伪”服务？这是由于我们还没有和真正的后端 API 对接，我们暂时写的服务，这些服务里面大多是自己生成一些数据返回，所以我们称之为“伪”服务

还是使用 CLI 生成服务的基础代码，既然是 `auth` 中的服务，我们就为服务在 `auth` 下新建一个 `services` 文件夹。然后键入下面的命令，注意 `-m` 就是 `--module` 的缩写形式，指定这个参数将在对应的模块文件中的 `providers` 数组中自动加上这个生成的服务：

```bash
ng g s auth/services/auth -m auth/auth.module.ts
```

这会为我们生成 `AuthService` 和它的测试文件。打开 `src/app/auth/services/auth.service.ts` ，将其改造成下面的样子。

```ts
import { Injectable } from '@angular/core';
import { Observable, of } from 'rxjs';
import { AuthModule } from '../auth.module';
import { Auth } from '../../domain/auth';
import { User } from '../../domain/user';

@Injectable()
export class AuthService {
  constructor() {}
  /**
   * 用于用户的登录鉴权
   * @param auth 用户的登录信息，一般是登录名（目前是用户名，以后会允许手机号）和密码
   */
  login(auth: Auth): Observable<User> {
    const user: User = {
      username: 'test',
      mobile: '13012341234',
      email: 'zhangsan@local.dev',
      name: 'Zhang San',
      avatar: 'assets/img/avatar/001.svg',
      roles: []
    };
    return of(user);
  }
  /**
   * 用于用户的注册
   * @param user 用户注册信息
   */
  register(user: User): Observable<User> {
    const user_add: User = { ...user, id: '123abc' };
    return of(user_add);
  }
  /**
   * 请求发送短信验证码到待验证手机，成功返回空对象 {}
   * @param mobile 待验证的手机号
   */
  requestSmsCode(mobile: string) {
    return of({});
  }
  /**
   * 验证手机号和短信验证码是否匹配
   * @param mobile 待验证手机号
   * @param code 收到的短信验证码
   */
  verifySmsCode(mobile: string, code: string) {
    return of(mobile === '13012341234' && code === '123456');
  }
  /**
   * 检查用户名是否唯一
   * @param username 用户名
   */
  checkUniqueUsername(username: string) {
    return of(username === 'lisi');
  }
  /**
   * 检查电子邮件是否唯一
   * @param email 电子邮件
   */
  checkUniqueEmail(email: string) {
    return of(email === 'lisi@local.dev');
  }
  /**
   * 检查手机号是否唯一
   * @param mobile 手机号
   */
  checkUniqueMobile(mobile: string) {
    return of(mobile === '13112341234');
  }
}
```

这个文件中我们使用了 `rxjs` 中的 `Observable` 作为返回值，这个读者可以先不用试图深入理解，把它看成一个异步的类型（类似 `Promise` ）即可，后面对于 `rxjs` 我们会有较详细的阐述。

类似的，我们为“佳句”生成对应服务

```bash
ng g s auth/services/quote -m auth/auth.module.ts
```

同样更改其生成的文件 `src/app/auth/services/quote.service.ts` 为下面的样子，这个服务中，我们手动构造了一个“佳句”数组，然后在 `getQuotes` 中返回：

```ts
import { Injectable } from '@angular/core';
import { AuthModule } from '../auth.module';
import { of, Observable } from 'rxjs';
import { Quote } from '../../domain/quote';

@Injectable()
export class QuoteService {
  constructor() {}

  getQuotes(): Observable<Quote[]> {
    const quotes: Quote[] = [
      {
        cn:
          '我突然就觉得自己像个华丽的木偶,演尽了所有的悲欢离合,可是背上总是有无数闪亮的银色丝线,操纵我哪怕一举手一投足。',
        en:
          'I suddenly feel myself like a doll,acting all kinds of joys and sorrows.There are lots of shining silvery thread on my back,controlling all my action.',
        imgUrl: '/assets/img/quotes/0.jpg'
      },
      {
        cn:
          '被击垮通常只是暂时的，但如果你放弃的话，就会使它成为永恒。（Marilyn vos Savant）',
        en:
          'Being defeated is often a temporary condition. Giving up is what makes it permanent.',
        imgUrl: '/assets/img/quotes/1.jpg'
      },
      {
        cn: '不要只因一次挫败，就放弃你原来决心想达到的梦想。（莎士比亚）',
        en:
          'Do not, for one repulse, forgo the purpose that you resolved to effect.',
        imgUrl: '/assets/img/quotes/2.jpg'
      },
      {
        cn: '想有发现就要实验，这项实验需要时间。—《神盾局特工》',
        en:
          'Discovery requires experimentation, and this experiment will take time.',
        imgUrl: '/assets/img/quotes/3.jpg'
      },
      {
        cn:
          '这世界并不会在意你的自尊，这世界希望你在自我感觉良好之前先要有所成就。',
        en:
          "The world won't care about your self-esteem. The world will expect you to accomplish something before you feel good about yourself.",
        imgUrl: '/assets/img/quotes/4.jpg'
      },
      {
        cn: '当你最终放开了过去，更好的事就会到来。',
        en:
          'When you finally let go of the past, something better comes along.',
        imgUrl: '/assets/img/quotes/5.jpg'
      },
      {
        cn:
          '我们学着放开过去伤害我们的人和事，学着只向前看。因为生活本来就是一往直前的。',
        en:
          'We learn to let go of things and people that hurt us in the past and just move on. For life is all about moving on.',
        imgUrl: '/assets/img/quotes/6.jpg'
      },
      {
        cn:
          '绝不要因为怕辛苦而拒绝一个想法、梦想或是目标，成功的路上难免伴随辛苦。（Bob Proctor）',
        en:
          'Never reject an idea, dream or goal because it will be hard work. Success rarely comes without it.',
        imgUrl: '/assets/img/quotes/7.jpg'
      },
      {
        cn:
          '我们在人生中会作出许多选择，带着这些选择继续生活，才是人生中最难的一课。《妙笔生花》',
        en:
          'We all make our choices in life. The hard thing to do is live with them.',
        imgUrl: '/assets/img/quotes/8.jpg'
      },
      {
        cn:
          '我总是对新的一天充满喜悦，这是一次新的尝试、一个新的开始，翘首以待，黎明之后或是惊喜。（约翰·博因顿·普里斯特利）',
        en:
          'I have always been delighted at the prospect of a new day, a fresh try, one more start, with perhaps a bit of magic waiting somewhere behind the morning.',
        imgUrl: '/assets/img/quotes/9.jpg'
      }
    ];
    return of(quotes);
  }
}
```

有了“笨组件”，有了服务，我们就可以构建“聪明”组件了。

## “聪明组件”的构建

有了“笨组件”，聪明组件就非常简单了。我们在 `auth` 目录下新建 `containers` 子目录。

### 登录页面

使用下面命令生成基础文件

```bash
ng g c auth/containers/login --module auth/auth.module.ts
```

然后更改 `src/app/auth/containers/login/login.component.html` ，使用我们前面创建的“佳句”和“登录表单”两个“笨组件”，并设置其对应的属性和处理其产生的事件。这里面你又会看到一些奇怪的符号，不要紧，我们稍后会讲。

```html
<div fxLayout="row" fxLayout.xs="column" fxLayoutAlign="center space-around">
  <app-quote [chineseQuote]="(quote$ | async)?.cn" [englishQuote]="(quote$ | async)?.en" [quoteImg]="(quote$ | async)?.imgUrl">
  </app-quote>
  <app-login-form (submitEvent)="processLogin($event)"></app-login-form>
</div>
```

更改 `src/app/auth/containers/login/login.component.ts` 为

```ts
import { Component } from '@angular/core';
import { Observable } from 'rxjs';
import { map, take } from 'rxjs/operators';

import { QuoteService } from '../../services/quote.service';
import { Quote } from '../../../domain/quote';
import { Auth } from '../../../domain/auth';
import { AuthService } from '../../services/auth.service';

@Component({
  selector: 'login',
  templateUrl: './login.component.html',
  styleUrls: ['./login.component.scss']
})
export class LoginComponent {
  quote$: Observable<Quote>;
  constructor(
    private quoteService: QuoteService,
    private authService: AuthService
  ) {
    this.quote$ = quoteService
      .getQuotes()
      .pipe(map(quotes => quotes[Math.floor(Math.random() * 10)]));
  }
  processLogin(auth: Auth) {
    this.authService
      .login(auth)
      .pipe(take(1))
      .subscribe(u => console.log(u));
  }
}
```

在这个“聪明”组件的构造函数中，我们注入了两个服务，分别是负责“佳句”的 `QuoteService` 和负责鉴权的 `AuthService` 。

我们还定义了一个成员变量 `quote$: Observable<Quote>;` ，我们在 `QuoteService` 中的 `getQuotes` 方法中得到的是一个数组 `Quote[]` ，但是在组件中我们显然需要的是一个数组中的元素。所以我们的成员变量的定义是一个 `Quote` 类型，而不是 `Quote[]` 。

但是我们既然服务中返回的是一个数组，怎么变成一个元素呢？这就是 `map(quotes => quotes[Math.floor(Math.random() * 10)])` 这句起的作用，它将一个数组变换成一个元素，具体就是得到数组 `quotes` 后，返回一个随机位置的元素 `Math.floor(Math.random() * 10)` 。这样我们每次进入这个页面都会显示不同的“佳句”。

对于 `Observable` 类型的变量我们有两种方式取得其订阅值。

其一就是类似 `quote$` 的处理中，我们并不直接订阅，而是在模版中使用 `quote$ | async` 。使用 `async` 管道的好处在于不用显式的订阅和销毁，Angular 会帮你完成这一切。

第二种就是显式订阅，就像我们在 `processLogin` 方法中处理的那样 `subscribe();` 。但是这种显式订阅的时候我们一定要注意内存泄漏问题，能显式销毁尽量显式销毁。这个例子中我们没有做销毁动作的原因是使用了 `take(1)` ，这意味着收到一个数据后就完成了这个流的订阅，订阅也就自动销毁了。这一块还是看的云里雾里的话也没关系，我们在讲 `rxjs` 时会再讨论这个点的。

### 注册页面

使用下面命令生成基础文件

```bash
ng g c auth/containers/register --module auth/auth.module.ts
```

更改其模版以使用我们的注册表单“笨组件”，并设置几个异步验证器属性 `[usernameValidator]="usernameValidator"` 、 `[emailValidator]="emailValidator"` 、 `[mobileValidator]="mobileValidator"` 和处理其表单提交事件 `(submitEvent)="processRegister($event)"`

```html
<div fxLayout="row" fxLayout.xs="column" fxLayoutAlign="center space-around">
  <register-form (submitEvent)="processRegister($event)" [usernameValidator]="usernameValidator" [emailValidator]="emailValidator" [mobileValidator]="mobileValidator">
  </register-form>
</div>
```

在其组件文件中，我们在其构造函数中构建了几个异步验证器，由于在 Angular 中属性的绑定是在 `ngOnChanges` 时做，而构造函数早于任何一个生命周期钩子，所以我们在构造时进行这个操作，可以让后面的数据绑定正确的进行。

```ts
import { Component } from '@angular/core';
import { AsyncValidatorFn } from '@angular/forms';
import { take } from 'rxjs/operators';
import { Router } from '@angular/router';
import { User } from '../../../domain/user';
import { AuthService } from '../../services/auth.service';
import { RegisterValidator } from '../../validators/register.validator';

@Component({
  selector: 'register',
  templateUrl: './register.component.html',
  styleUrls: ['./register.component.scss']
})
export class RegisterComponent {
  usernameValidator: AsyncValidatorFn;
  emailValidator: AsyncValidatorFn;
  mobileValidator: AsyncValidatorFn;
  constructor(private service: AuthService, private router: Router) {
    this.usernameValidator = RegisterValidator.validateUniqueUsername(service);
    this.mobileValidator = RegisterValidator.validateUniqueMobile(service);
    this.emailValidator = RegisterValidator.validateUniqueEmail(service);
  }

  processRegister(user: User) {
    this.service
      .register(user)
      .pipe(take(1))
      .subscribe(u => {
        console.log(u);
      });
  }
}
```

我们把使用到的异步验证器抽取到另一个单独文件中 `src/app/auth/validators/register.validator.ts` ，新建了一个 `RegisterValidator` 类和几个静态方法返回对应的异步验证器。

```ts
import { AbstractControl } from '@angular/forms';
import { of } from 'rxjs';
import { map } from 'rxjs/operators';
import { AuthService } from '../services/auth.service';

export class RegisterValidator {
  static validateUniqueUsername(service: AuthService) {
    return (control: AbstractControl) => {
      const val = control.value;
      if (!val) {
        return of(null);
      }
      return service.usernameExisted(val).pipe(
        map(res => {
          return res.existed ? { usernameNotUnique: true } : null;
        })
      );
    };
  }
  static validateUniqueEmail(service: AuthService) {
    return (control: AbstractControl) => {
      const val = control.value;
      if (!val) {
        return of(null);
      }
      return service.emailExisted(val).pipe(
        map(res => {
          return res.existed ? { emailNotUnique: true } : null;
        })
      );
    };
  }
  static validateUniqueMobile(service: AuthService) {
    return (control: AbstractControl) => {
      const val = control.value;
      if (!val) {
        return of(null);
      }
      return service.mobileExisted(val).pipe(
        map(res => {
          return res.existed ? { mobileNotUnique: true } : null;
        })
      );
    };
  }
}
```

## 路由处理

如我们之前所说，每个模块有自己的路由，`auth` 模块在生成的时候已经生成了自己的路由模块 -- `src/app/auth/auth-routing.module.ts`

我们需要把这个路由模块中路由数组改造一下，添加我们的路由。

```ts
import { NgModule } from '@angular/core';
import { Routes, RouterModule } from '@angular/router';
import { LoginComponent } from './containers/login/login.component';
import { RegisterComponent } from './containers/register/register.component';
import { ForgotPasswordComponent } from './containers/forgot-password/forgot-password.component';

const routes: Routes = [
  {
    path: 'auth',
    redirectTo: 'auth/login',
    pathMatch: 'full'
  },
  {
    path: 'auth/login',
    component: LoginComponent
  },
  {
    path: 'auth/register',
    component: RegisterComponent
  },
  {
    path: 'auth/forgot',
    component: ForgotPasswordComponent
  }
];

@NgModule({
  imports: [RouterModule.forChild(routes)],
  exports: [RouterModule]
})
export class AuthRoutingModule {}
```

当然别忘了同样更新根路由 -- `src/app/core/app-routing.module.ts` ，我们将默认路径指向 `/auth`

```ts
import { NgModule } from '@angular/core';
import { RouterModule, Routes } from '@angular/router';
import { PageNotFoundComponent } from './components/page-not-found.component';

const routes: Routes = [
  {
    path: '',
    redirectTo: '/auth',
    pathMatch: 'full'
  },
  {
    path: '**',
    component: PageNotFoundComponent
  }
];

@NgModule({
  imports: [RouterModule.forRoot(routes)],
  exports: [RouterModule]
})
export class AppRoutingModule {}
```

最后，由于我们还没有实现懒加载，还需要把 `auth` 模块导入到 `CoreModule` 中

```ts
// 省略
import { AuthModule } from '../auth/auth.module';

@NgModule({
  declarations: [
    // 省略
  ],
  imports: [
    SharedModule,
    HttpClientModule,
    AuthModule, //<------------这里
    AppRoutingModule,
    BrowserAnimationsModule
  ]
})
export class CoreModule {
  // 省略
}
```
