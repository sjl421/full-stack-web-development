# 完成忘记密码功能

## 使用 rxjs 打造短信验证码控件

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
      tap(_ => {
        this.requestCode.emit();
      }),
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

## 忘记密码向导组件

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
