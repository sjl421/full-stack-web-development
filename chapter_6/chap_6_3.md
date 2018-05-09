# Http 拦截

大家这几章来回在 Angular 和 Spring 之间切换，是否也感觉到 Angular 和 Spring 之间有很多共享的概念，比如依赖注入、比如注解。Http Interceptor 这个概念对于 Angular 也是存在的。前端是要发起 HTTP 请求到后端的，那么如果我们需要，比如统一在前端发起的请求中加入某些指定的 Header ，这个当然可以在每个服务中去实现。但是不是有点繁琐啊，而且也不利于代码的重构，因为这些加 Header 的地方分散在程序的各个地方。当然 Interceptor 能干的事情不止是这些，我们也可以截获返回的 Response ，根据不同的返回状态做特定的处理。

## 实现一个简单的 HttpInterceptor

我们首先来实现一个 interceptor ，当客户端发送 API 请求到服务器的时候，如果这个 API 的路径传递有误，那么后端会返回一个 `404` ，我们这个截断器的作用就是拦截到请求后，如果返回的错误码是 404 的话，就导航到前端的 404 路由。

```ts
import { Injectable, Injector } from '@angular/core';
import { Router } from '@angular/router';
import {
  HttpInterceptor,
  HttpRequest,
  HttpHandler,
  HttpEvent
} from '@angular/common/http';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';

@Injectable()
export class NotFoundInterceptor implements HttpInterceptor {
  constructor(private _injector: Injector) {}

  intercept(
    req: HttpRequest<any>,
    next: HttpHandler
  ): Observable<HttpEvent<any>> {
    return next.handle(req).pipe(
      tap(
        event => {},
        err => {
          if (err.status === 404) {
            const router = this._injector.get(Router);
            router.navigate(['/404']);
          }
        }
      )
    );
  }
}
```

很常见的一个场景是当用户浏览某个不存在的链接时，系统会提供一个 404 页面。传统的 web 开发中，这一块后端会统一处理。但在前后端分离的场景中，我们要知道 Angular 生成的是一个单页面应用 (最终只有一个 `index.html` )，所谓的路由其实只是用来装载不同的组件，浏览器地址栏的那个路由链接其实只是显示而已。所以在我们的场景中，只需要建立一个 404 的组件，在恰当的时候显示它就好了。
