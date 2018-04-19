# 完成忘记密码前端设计

忘记密码其实是一个相对复杂的控件，因为里面涉及到 2 个大的步骤，第一个是验证手机的过程，第二个是设置新密码的过程，第一个步骤成功的前提下才能完成第二个。和前面的注册、登录等不太一样，我们不是一个表单就处理完成了。所以需要一个向导组件，还好 `Angular Material` 提供了内建的支持： `Stepper` 。

![忘记密码向导组件](/assets/2018-04-10-19-09-27.png)

除了这个向导，验证手机也是一个相对复杂的控件，这个控件虽然看起来很简单，但是内部的逻辑并不是很 easy 。

## 使用 rxjs 打造短信验证码控件

首先还是 CLI 生成短信验证码组件基础文件，由于这个控件可能会在多处使用到，我们把它创建在 `shared` 目录中。

```bash
ng g c shared/components/verify-mobile
```

打开 `src/app/shared/components/verify-mobile/verify-mobile.component.html` ，将其改造成下面的样子。

```html
<div [formGroup]="form">
  <mat-form-field class="full-width">
    <input matInput placeholder="注册时使用的手机号" formControlName="regMobile">
    <mat-error> {{ mobileErrors }} </mat-error>
  </mat-form-field>
  <mat-form-field class="full-width">
    <input matInput type="text" [placeholder]="codePlaceholder" formControlName="smsCode">
    <button mat-button matSuffix #veriBtn [disabled]="(btnLabel$ | async).indexOf('发送') === -1 || form.get('regMobile').errors !== null">
      {{ btnLabel$ | async }} </button>
    <mat-error> 验证码不正确 </mat-error>
  </mat-form-field>
</div>
```

这个组件的 UI 表现形式还是比较简单的，只有两个输入框，上面的是手机号的输入框，下面的是验证码的输入框，但是下面这个输入框的右侧会有一个按钮。这个按钮的状态相对复杂：

* 文字状态：一开始时文字是“发送”，点击后，文字变成了一个 60 秒倒计时，倒计时结束后，按钮文字变成“再次发送”
* 禁用/可用状态：在手机号不合法的情况下，按钮应是禁用状态，手机号合法时可用；在点击“发送”后，60 秒倒计时期间，按钮禁用，倒计时结束又变成可用。

此外由于这是一个“笨组件”，所以它并不负责真正的验证码的发送或校检。但是如果交给外部使用它的父组件处理的话，我们是要给出一些输出型属性（事件）让外部可以得到相关的信息。

1. 手机号：输入的手机号在验证合法后会产生一个事件，将手机号发送给事件处理者。
2. 发送按钮的点击事件应该暴露给外部，以便父组件调用验证码 API 请求发送短信。

再有一个地方就是这个组件如果不是在注册页面使用的时候，比如是在更改个人信息页面的“重置密码”功能中使用时，我们是不希望用户可以更改手机号的，因为这个手机号就是用户注册的手机号，如果可以更改，那任何人都可以更改密码了。所以我们给组件设置了 `@Input() mobile: string | null = null;` 属性，如果设置了 `非 null` 的手机号，那么手机号这个输入框就禁用了。

这个表单控件的值是一个对象 `{mobile: string; code: string;}` ，外部取得值之后可以自行验证手机号和短信验证码是否匹配。

```ts
import {
  Component,
  OnInit,
  forwardRef,
  ChangeDetectionStrategy,
  EventEmitter,
  Output,
  Input,
  ViewChild,
  ElementRef,
  OnDestroy
} from '@angular/core';
import {
  NG_VALUE_ACCESSOR,
  NG_VALIDATORS,
  Validators,
  FormControl,
  ControlValueAccessor,
  FormBuilder,
  FormGroup
} from '@angular/forms';
import { interval, fromEvent, Observable, Subscription } from 'rxjs';
import {
  map,
  startWith,
  takeWhile,
  switchMap,
  tap,
  debounceTime,
  filter
} from 'rxjs/operators';
import { mobilePattern } from '../../../utils/regex';
import { mobileErrorMsg } from '../../../utils/validate-errors';

@Component({
  selector: 'app-verify-mobile',
  templateUrl: './verify-mobile.component.html',
  styleUrls: ['./verify-mobile.component.scss'],
  providers: [
    {
      provide: NG_VALUE_ACCESSOR,
      useExisting: forwardRef(() => VerifyMobileComponent),
      multi: true
    },
    {
      provide: NG_VALIDATORS,
      useExisting: forwardRef(() => VerifyMobileComponent),
      multi: true
    }
  ],
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class VerifyMobileComponent
  implements ControlValueAccessor, OnInit, OnDestroy {
  @Input() mobilePlaceholder = '绑定手机号';
  @Input() codePlaceholder = '请输入短信验证码';
  @Input() countdown = 60;
  @Input() mobile: string | null = null;
  @Output() requestCode = new EventEmitter<string>();
  @Output() mobileInputEvent = new EventEmitter<string>();
  @ViewChild('veriBtn', { read: ElementRef })
  veriBtn: ElementRef;
  btnLabel$: Observable<string>;
  form: FormGroup;
  private subs: Subscription[] = [];
  private propagateChange = (_: any) => {};

  constructor(private fb: FormBuilder) {}

  ngOnInit() {
    this.form = this.fb.group({
      regMobile: [
        { value: this.mobile, disabled: this.mobile },
        Validators.compose([
          Validators.required,
          Validators.pattern(mobilePattern)
        ])
      ],
      smsCode: [
        '',
        Validators.compose([Validators.required, Validators.pattern(/^\d{6}$/)])
      ]
    });
    if (!this.mobile) {
      const mobile = this.form.get('regMobile');
      this.subs.push(
        mobile.valueChanges
          .pipe(filter(_ => mobile.errors === null))
          .subscribe(val => this.mobileInputEvent.emit(val))
      );
    }
    const smsCode = this.form.get('smsCode');
    const countDown$ = interval(1000).pipe(
      map(i => this.countdown - i),
      takeWhile(v => v >= 0),
      startWith(this.countdown)
    );

    this.btnLabel$ = fromEvent(this.veriBtn.nativeElement, 'click').pipe(
      tap(_ => this.requestCode.emit()),
      switchMap(_ => countDown$),
      map(i => (i > 0 ? `还剩 ${i} 秒` : `再次发送`)),
      startWith('发送')
    );

    if (smsCode) {
      const code$ = smsCode.valueChanges;
      this.subs.push(
        code$.pipe(debounceTime(400)).subscribe(v =>
          this.propagateChange({
            mobile: this.mobile
              ? this.mobile
              : this.form.get('regMobile').value,
            code: v
          })
        )
      );
    }
  }
  ngOnDestroy(): void {
    this.subs.forEach(sub => {
      if (sub) {
        sub.unsubscribe();
      }
    });
    this.subs = [];
  }
  // 验证表单，验证结果正确返回 null 否则返回一个验证结果对象
  validate(c: FormControl): { [key: string]: any } | null {
    const val = c.value;
    if (!val) {
      return null;
    }
    const smsCode = this.form.get('smsCode');
    if (smsCode) {
      return smsCode.valid
        ? null
        : {
            smsCodeInvalid: true
          };
    }
    return {
      smsCodeInvalid: true
    };
  }
  writeValue(obj: any): void {
    this.mobile = obj;
  }
  registerOnChange(fn: any): void {
    this.propagateChange = fn;
  }
  registerOnTouched(fn: any): void {}

  get mobileErrors() {
    const mobile = this.form.get('regMobile');
    if (!mobile) {
      return '';
    }
    return mobileErrorMsg(mobile);
  }
}
```

这个组件是个表单组件，所以相关的内容我们就不赘述了，前面有较为详细的分析。这里我们主要来看一下是如何用 `rxjs` 解决逻辑问题的。

这个组件的页面上有以下的事件/数据流

* `mobile.valueChanges` -- 手机号输入产生的值的数据流。
  * 这个流应用 `filter` 操作符过滤掉不合法的手机号，然后在订阅中发送手机号 `this.mobileInputEvent.emit(val)`
* `countDown$` -- 倒计时的数据流。
  * 首先由 `interval` 每秒自动产生一个顺序增长序列（`0, 1, 2, 3, ...`）
  * 然后使用 `map` 操作符进行变换： `map(i => this.countdown - i)` ，就是用 60 减去 `interval` 得到的增长序列，这样就变换成一个顺序减少的序列。
  * 倒计时应该到 0 就结束了，所以我们使用 `takeWhile(v => v >= 0)` 让其到 0 后自动完成。
  * `interval` 是等到时间间隔后才发送值的，所以我们给出初始值 `startWith(this.countdown)` 。
* `btnLabel$` -- 按钮标签文字的数据流。
  * 这个流最开始是由按钮点击事件产生的 `fromEvent(this.veriBtn.nativeElement, 'click')`
  * 点击之后我们应该立即发送事件给外部 `tap(_ => this.requestCode.emit())`
  * 然后切换到倒计时流并执行高阶到低阶的转换 `switchMap(_ => countDown$)` ，此时流转换成了倒计时数字的数据流。
  * 再通过 `map` 将数字流转换成按钮要显示的文字 ``map(i => (i > 0 ? `还剩 ${i} 秒` : `再次发送`))``
  * 别忘了给这个流一个初始值，否则刚进入页面是，没有点击事件，这个流就没有数据，这样按钮文字就没有了。因此我们使用 `startWith('发送')`
* `smsCode.valueChanges` -- 短信验证码输入产生的值的数据流。
  * 这个流我们使用了 `debounceTime(400)` 做了节流处理，也就是 `400ms` 内发生的事件都丢弃了，因为用户输入时可能发生错误，我们不想频繁的给父组件发送事件。
  * 在其订阅方法中，我们将手机号和验证码构成一个对象发射出去了。

## 忘记密码向导“笨组件”

短信验证码表单控件完成后，我们就可以把它用在我们的“忘记密码向导”中了，当然一开始还是要创建这个组件。

```bash
ng g c auth/components/forgot-password-form -m auth/auth.module.ts
```

然后就开始设计这个向导了，这里面最重要的就是 `Stepper` 组件。 `Angular Material` 提供 `mat-horizontal-stepper` 和 `mat-vertical-stepper` 两个向导的父容器，分别设置水平排列的步骤或垂直排列的步骤。在父容器内，使用 `mat-step` 定义一个步骤。

父容器中可以指定 `linear` 属性，如果为 `true` ，就是必须一步一步执行，不可以点击跳到其他步骤，如果为 `false` 当然就允许。

`mat-step` 中的 `stepControl` 属性是要指定一个 `AbastractControl` ，也就是 `FormControl` 或 `FormGroup` ，这个 `AbastractControl` 的验证状态决定了这个步骤是否完成。使用多个 `Form` 表单配合 `Stepper` 是一个常见的做法，但也可以单个表单对应多个步骤，甚至可以不用表单。这里就不展开讲了，有兴趣的同学可以去官网了解详情 <https://material.angular.io/components/stepper/overview>

```html
<mat-horizontal-stepper [linear]="true">
  <mat-step [stepControl]="mobileForm">
    <form [formGroup]="mobileForm" (ngSubmit)="submit(mobileForm, $event)">
      <ng-template matStepLabel> 验证绑定手机 </ng-template>
      <app-verify-mobile formControlName="oldCode" [mobile]="mobile" [mobilePlaceholder]="'绑定手机'" (requestCode)="makeRequest($event)"
        (mobileInputEvent)="mobileInput($event)">
      </app-verify-mobile>
      <div>
        <button mat-button matStepperNext> 下一步 </button>
      </div>
    </form>
  </mat-step>
  <mat-step [stepControl]="newPasswordForm">
    <form [formGroup]="newPasswordForm" (ngSubmit)="submit(newPasswordForm, $event)">
      <ng-template matStepLabel>设置新密码</ng-template>
      <mat-form-field class="full-width">
        <input matInput placeholder="请输入新密码" type="password" formControlName="password">
      </mat-form-field>
      <mat-form-field class="full-width">
        <input matInput placeholder="请再次输入" type="password" formControlName="repeat">
        <mat-error> 两次密码输入不一致 </mat-error>
      </mat-form-field>
      <div>
        <button mat-button matStepperNext> 下一步 </button>
      </div>
    </form>
  </mat-step>
  <mat-step>
    <ng-template matStepLabel> 完成 </ng-template>
    密码修改成功。
    <div>
      <button mat-button matStepperPrevious>返回</button>
    </div>
  </mat-step>
</mat-horizontal-stepper>
```

接下来看组件文件，由于有两个表单，所以我们在 `ngOnInit` 中需要分别初始化。为了验证手机和短信验证码是否匹配，我们需要一个异步验证器，但这个我们会交给聪明组件处理，所以这里设置了一个 `@Input() codeValidator: ValidatorFn;` 。

还需要指出的是，第二个表单同样需要两个密码比较，但这里我们并没有采用前面注册表单组件中的做法，而是给出了另一种方式。这里没有在 `FormGroup` 级别去做验证器，而是给 `repeat` 做了个 `FormControl` 的验证器。但比较的思路是基本一致的，这里不过是给大家展示另一种方式： `control.parent.get(otherCtrlName)` 。请注意在真实的开发过程中，如果遇到一种验证方式是多处需要的时候，最好将验证器剥离出来，或者抽象该组件为公共组件。

```ts
import { Component, OnInit, Input, Output, EventEmitter } from '@angular/core';
import {
  ValidatorFn,
  FormGroup,
  FormBuilder,
  Validators,
  FormControl
} from '@angular/forms';

@Component({
  selector: 'forgot-password-form',
  templateUrl: './forgot-password-form.component.html',
  styleUrls: ['./forgot-password-form.component.scss']
})
export class ForgotPasswordFormComponent implements OnInit {
  @Input() mobile: string | null = null;
  @Input() codeValidator: ValidatorFn;
  @Output() submitPassword = new EventEmitter<string>();
  @Output() requestCode = new EventEmitter<string>();
  @Output() mobileInputEvent = new EventEmitter<string>();
  mobileForm: FormGroup;
  newPasswordForm: FormGroup;

  constructor(private fb: FormBuilder) {}

  ngOnInit() {
    this.mobileForm = this.fb.group({
      oldCode: [this.mobile, Validators.required, this.codeValidator]
    });

    this.newPasswordForm = this.fb.group({
      password: [
        '',
        Validators.compose([Validators.required, Validators.minLength(8)])
      ],
      repeat: ['', [Validators.required, this.matchPassword('password')]]
    });
  }

  submit(form: FormGroup, ev: Event) {
    if (!form.valid || !form.value) {
      return;
    }
    if (form.value.password) {
      this.submitPassword.emit(form.value.password);
    }
  }

  makeRequest(mobile: string) {
    this.requestCode.emit(mobile);
  }

  matchPassword(otherCtrlName: string) {
    let thisControl: FormControl;
    let otherControl: FormControl;
    return (control: FormControl) => {
      if (!control.parent) {
        return null;
      }
      // Initializing the validator.
      if (!thisControl) {
        thisControl = control;
        otherControl = control.parent.get(otherCtrlName) as FormControl;
        if (!otherControl) {
          throw new Error('matchPassword(): 未发现表单中有要比较的控件');
        }
        otherControl.valueChanges.subscribe(() => {
          thisControl.updateValueAndValidity();
        });
      }
      if (!otherControl) {
        return null;
      }
      if (otherControl.value !== thisControl.value) {
        return {
          matchOther: true
        };
      }
      return null;
    };
  }
  mobileInput(mobile: string) {
    this.mobileInputEvent.emit(mobile);
  }
}
```

## 忘记密码的聪明组件

最麻烦的部分都已经解决了，接下来就是把忘记密码的聪明组件完成，这就简单多了。

模版部分还是利用卡片组织，把忘记密码的笨组件 `forgot-password-form` 嵌入 `mat-card-content` 之中。卡片尾部提供链接可以返回登录或注册页面。

```html
<mat-card>
  <mat-card-header>
    <mat-card-title>
      <span> 更改密码 </span>
    </mat-card-title>
    <mat-card-subtitle>
      通过手机验证码更改您的密码
    </mat-card-subtitle>
  </mat-card-header>
  <mat-card-content>
    <forgot-password-form [codeValidator]="codeValidator" (codeRequestEvent)="processCodeRequest($event)" (mobileInputEvent)="processMobile($event)"
      (passwordEvent)="processPassword($event)">
    </forgot-password-form>
  </mat-card-content>
  <mat-card-actions>
    <div fxLayout="row" fxLayoutAlign="end stretch">
      <a mat-button routerLink="/auth/login"> {{ loginBtnText }} </a>
      <a mat-button routerLink="/auth/register"> {{ registerBtnText }} </a>
    </div>
  </mat-card-actions>
</mat-card>
```

组件文件中，在构造函数中注入 `AuthService` ，同时在构造函数中把笨组件需要的异步验证器构建好。 ``

```ts
import { Component, OnInit, Input } from '@angular/core';
import { AsyncValidatorFn } from '@angular/forms';
import { AuthService } from '../../services/auth.service';
import { SmsValidator } from '../../validators/sms.validator';
import { take } from 'rxjs/operators';

@Component({
  selector: 'forgot-password',
  templateUrl: './forgot-password.component.html',
  styleUrls: ['./forgot-password.component.scss']
})
export class ForgotPasswordComponent implements OnInit {
  loginBtnText = '登录';
  registerBtnText = '注册';
  codeValidator: AsyncValidatorFn;

  constructor(private service: AuthService) {
    this.codeValidator = SmsValidator.validateSmsCode(service);
  }

  ngOnInit() {}

  processCodeRequest(mobile: string) {
    this.service
      .requestSmsCode(mobile)
      .pipe(take(1))
      .subscribe(val => console.log(val));
  }

  processMobile(mobile: string) {
    console.log(mobile);
  }

  processPassword(password: string) {
    console.log(password);
  }
}
```

注意到，上面的异步验证器是抽象成一个独立的类 `SmsValidator` ，文件路径是 `src/app/auth/validators/sms.validator.ts`

```ts
import { AbstractControl } from '@angular/forms';
import { of } from 'rxjs';
import { map } from 'rxjs/operators';
import { AuthService } from '../services/auth.service';

export class SmsValidator {
  static validateSmsCode(service: AuthService) {
    return (control: AbstractControl) => {
      const val = control.value;
      if (!val.mobile || !val.code) {
        throw new Error('SmsValidator: 没有找到手机号或验证码');
      }
      return service.verifySmsCode(val.mobile, val.code).pipe(
        map(res => {
          return res ? null : { codeInvalid: true };
        })
      );
    };
  }
}
```

- [ ] Stepper 部分需要扩展一些
