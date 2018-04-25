# 响应式的 Http API 处理

后端的 API 既然已经搭好了，那么前端就需要调用了。但是这时的问题往往也伴随而来，由于 HTTP 访问一般要使用异步方式（如果同步的话，想想用户体验吧），而采用异步之后，很多时候会产生或者逻辑错误或者回调地狱。这个时候也就是体验 `rxjs` 优越性的时候了。

## Angular 中的 HTTP 服务

要使用 Angular 中的 Http 服务，一般有两个选择，如果在 Angular 4.x 时可以使用传统的位于 `@angular/http` 中的 `HttpModule` 。但在 Angular 4.3 之后，官方推荐使用位于 `@angular/common/http` 中的 `HttpClientModule` 。我们这里采用的就是 `HttpClientModule` ，注意我们之所以把这个 Module 放在 `CoreModule` 中导入，是因为 `HttpClient` 是以依赖形式 `Provide` 出来的，我们在同一个应用没有必要初始化多个 `HttpClient` 。

```ts
// 省略其他导入
import { HttpClientModule } from '@angular/common/http'; // <-- 这里

@NgModule({
  declarations: [
    // 省略组件声明
  ],
  imports: [
    SharedModule,
    HttpClientModule, // <--- 这里
    AuthModule,
    AppRoutingModule,
    BrowserAnimationsModule
  ]
})
export class CoreModule {
  // 省略
}
```

还记得我们的后端的 API 地址吗？好像是 <http://localhost:8080/api/xxx> ，那么我们直接这么使用的话，你没有后端代码怎么办？以后正式部署了怎么办？种种问题都会接踵而来。这个其实和前一章我们在后端要解决不同环境下使用不同数据库的问题类似，都是一个开发环境配置问题。

## Angular 的开发环境配置

在使用 Angular CLI 生成工程时，如果你留意到了的话，会发现在 `src` 下面有一个 `environments` 子目录。

![CLI 自动创建的 environments 目录](/assets/2018-04-24-11-11-41.png)

`environment.ts` 是默认的环境配置，如果你在没有指定使用哪些配置的时候， Angular 会默认取这个文件中的值。

```ts
export const environment = {
  production: false,
  apiBaseUrl: 'http://localhost:8080/api/'
};
```

在上面的环境配置中，我们指定了一个环境变量 `apiBaseUrl` ，现在如果我们在 `environments/environment.prod.ts` 中，设置一个不同的 `apiBaseUrl` ，我们一起看看应用中使用这个环境变量时会发生什么。

```ts
export const environment = {
  production: true,
  apiBaseUrl: 'http://local.dev/api/'
};
```

### 使用环境变量

为方便起见，我们直接在 CoreModule 中的构造函数中打印这个变量。

```ts
// 省略
import { environment } from '../../environments/environment'; // <--- 这里

@NgModule({
  declarations: [
    // 省略
  ],
  imports: [
    // 省略
  ]
})
export class CoreModule {
  constructor(
    @Optional()
    @SkipSelf()
    parentModule: CoreModule,
    ir: MatIconRegistry,
    ds: DomSanitizer
  ) {
    // 省略其他
    console.log(environment.apiBaseUrl); // <--- 这里
  }
}
```

然后使用 `ng serve` 启动应用，打开 Chrome 的开发者工具，观察控制台，可以看到现在打印出来的是 <http://localhost:8080/api/>

![Chrome 中打印 appBaseUrl](/assets/2018-04-24-11-38-37.png)

那么接下来我们使用 `ng serve -c production` 或者 `ng serve --prod` ，可以看到生产环境下的输出是 <http://local.dev/api/> 。

![在生产环境下的输出](/assets/2018-04-24-13-26-12.png)

命令中的 `production` 是怎么回事呢？看一下项目根目录的 `angular.json` ，我们会发现 `build` 和 `serve` 中都有 `configuration` ，里面定义了 `production` 这个配置。所以如果我们想有更多配置的时候，除了要再新建对应的 `environment.xxx.ts` 之外，还需要在这个 `angular.json` 中也定义一下配置的名称。而 `ng serve --prod` 其实是 CLI 在生产模式时自动切换到了 `production` 这个配置。

```json
{
  "$schema": "./node_modules/@angular-devkit/core/src/workspace/workspace-schema.json",
  "version": 1,
  "newProjectRoot": "projects",
  "projects": {
    "frontend": {
      "root": "",
      "projectType": "application",
      "prefix": "app",
      "architect": {
        "build": {
          // 省略其他设置
          "configurations": {
            "production": {
              "fileReplacements": [
                {
                  "replace": "src/environments/environment.ts",
                  "with": "src/environments/environment.prod.ts"
                }
              ],
              "optimization": true,
              "outputHashing": "all",
              "sourceMap": false,
              "extractCss": true,
              "namedChunks": false,
              "aot": true,
              "extractLicenses": true,
              "vendorChunk": false,
              "buildOptimizer": true
            }
          }
        },
        "serve": {
          "builder": "@angular-devkit/build-angular:dev-server",
          "options": {
            "browserTarget": "frontend:build"
          },
          "configurations": {
            "production": {
              "browserTarget": "frontend:build:production"
            }
          }
        },
        // 省略其他
      }
    },
    // 省略其他部分
  },
  "schematics": {
    "@schematics/angular:component": {
      "styleext": "scss"
    }
  }
}
```

### 让容器支持环境配置

其实在第二章，我们给出前端的容器镜像文件时已经支持了环境配置，我们在 `RUN npm run build -- --prod --configuration $env` 中已经指定了环境配置。

```Dockerfile
### STAGE 1: Build ###

# We label our stage as 'builder'
FROM node:8-alpine as builder

## 省略其他部分

## Build the angular app in production mode and store the artifacts in dist folder
ARG env=production
RUN npm run build --prod --configuration $env

## 省略其他部分
```

我们在 Dockerfile 中定义了一个参数 `env` ，而这个配置名称可以通过参数形式传入 Docker 制作过程，再回顾我们的 `docker-compose.yml` ，我们在构建镜像时使用 `args` 传入了 `env`

```yml
version: '3'
services:
  nginx:
    build:
      context: .
      dockerfile: ./docker/nginx/Dockerfile
      args:
        - env=production # <--- 这里以参数形式传入
    container_name: nginx
    ports:
      - 80:80
```

## 在前端服务中使用 HttpClient

那么现在我们可以更改 `frontend/src/app/auth/services/auth.service.ts` 为下面的代码。代码中可以看到 Angular 消费 Rest API 是非常简单的，因为 Angular 中将 HttpClient 的方法也分成了 `get` 、 `put` 、 `patch` 、 `post` 和 `delete` ，对应不同的 Rest API 操作，我们使用不同的方法。我们可以通过在构造函数中参数注入的方式得到 HttpClient 的实例 `constructor(private http: HttpClient)`

```ts
import { Injectable } from '@angular/core';
import { HttpClient, HttpHeaders, HttpParams } from '@angular/common/http';
import { Observable, of } from 'rxjs';
import { pluck } from 'rxjs/operators';
import { AuthModule } from '../auth.module';
import { Auth } from '../../domain/auth';
import { User } from '../../domain/user';
import { environment } from '../../../environments/environment';
import { Captcha } from '../../domain/captcha';

@Injectable()
export class AuthService {
  private headers = new HttpHeaders().append(
    'Content-Type',
    'application/json'
  );
  constructor(private http: HttpClient) {}
  /**
   * 用于用户的登录鉴权
   * @param auth 用户的登录信息，一般是登录名（目前是用户名，以后会允许手机号）和密码
   */
  login(auth: Auth): Observable<string> {
    return this.http
      .post<{ id_token: string }>(
        `${environment.apiBaseUrl}auth/login`,
        JSON.stringify(auth),
        { headers: this.headers }
      )
      .pipe(pluck('id_token'));
  }
  /**
   * 用于用户的注册
   * @param user 用户注册信息
   */
  register(user: User): Observable<string> {
    return this.http
      .post<{ id_token: string }>(
        `${environment.apiBaseUrl}auth/register`,
        JSON.stringify(user),
        { headers: this.headers }
      )
      .pipe(pluck('id_token'));
  }
  /**
   * 请求发送短信验证码到待验证手机，成功返回空对象 {}
   * @param mobile 待验证的手机号
   */
  requestSmsCode(mobile: string, token: string): Observable<void> {
    const params = new HttpParams()
      .append('mobile', mobile)
      .append('token', token);
    return this.http.get<void>(`${environment.apiBaseUrl}auth/mobile`, {
      headers: this.headers,
      params: params
    });
  }
  /**
   * 验证手机号和短信验证码是否匹配
   * @param mobile 待验证手机号
   * @param code 收到的短信验证码
   */
  verifySmsCode(mobile: string, code: string): Observable<void> {
    return this.http.post<void>(
      `${environment.apiBaseUrl}auth/mobile`,
      JSON.stringify({ mobile: mobile, code: code }),
      { headers: this.headers }
    );
  }
  /**
   * 请求发送短信验证码到待验证手机，成功返回空对象 {}
   * @param mobile 待验证的手机号
   */
  requestCaptcha() {
    return this.http.get<Captcha>(`${environment.apiBaseUrl}auth/captcha`, {
      headers: this.headers
    });
  }
  /**
   * 验证手机号和短信验证码是否匹配
   * @param mobile 待验证手机号
   * @param code 收到的短信验证码
   */
  verifyCaptcha(token: string, code: string) {
    return this.http.post<{ validate_token: string }>(
      `${environment.apiBaseUrl}auth/captcha`,
      JSON.stringify({ captcha_token: token, captcha_code: code }),
      {
        headers: this.headers
      }
    );
  }
  /**
   * 检查用户名是否唯一
   * @param username 用户名
   */
  usernameExisted(username: string) {
    const params = new HttpParams().append('username', username);
    return this.http.get<{ existed: boolean }>(
      `${environment.apiBaseUrl}auth/search/username`,
      {
        headers: this.headers,
        params: params
      }
    );
  }
  /**
   * 检查电子邮件是否唯一
   * @param email 电子邮件
   */
  emailExisted(email: string) {
    const params = new HttpParams().append('email', email);
    return this.http.get<{ existed: boolean }>(
      `${environment.apiBaseUrl}auth/search/email`,
      {
        headers: this.headers,
        params: params
      }
    );
  }
  /**
   * 检查手机号是否唯一
   * @param mobile 手机号
   */
  mobileExisted(mobile: string) {
    const params = new HttpParams().append('mobile', mobile);
    return this.http.get<{ existed: boolean }>(
      `${environment.apiBaseUrl}auth/search/mobile`,
      {
        headers: this.headers,
        params: params
      }
    );
  }
}
```

HttpClient 都支持那些操作，有哪些方法？最好的办法是看看它的类定义，下面的代码列出了它主要方法。可以看到对于常见的 Rest API 操作来说， `delete` 、 `get` 方法是没有 body 参数的，而 `patch` 、 `put` 、 `post` 是有的，这个 `body` 就是 Http Request Body ，对于 `get` 和 `delete` 操作，都不会携带诸如 `json` 、 `xml` 等对象，而是直接在 `URL` 上体现参数。

```ts
class HttpClient {
  constructor(handler: HttpHandler)
  request(first: string | HttpRequest<any>, url?: string, options: {...}): Observable<any>
  delete(url: string, options: {...}): Observable<any>
  get(url: string, options: {...}): Observable<any>
  head(url: string, options: {...}): Observable<any>
  jsonp<T>(url: string, callbackParam: string): Observable<T>
  options(url: string, options: {...}): Observable<any>
  patch(url: string, body: any | null, options: {...}): Observable<any>
  post(url: string, body: any | null, options: {...}): Observable<any>
  put(url: string, body: any | null, options: {...}): Observable<any>
}
```

有的同学观察到了，很多方法带了一个 options 的参数，这个参数是用来指定一些请求的特殊配置，如果需要写入特定的 Header，那么需要使用 `HttpHeaders` ，如果需要使用参数（也就是 `/xxx?a=value1&b=value2` ）， 就得使用 HttpParams 。那么这个 options 的对象结构如下所示，这就可以理解我们上面代码中的诸如 `{ headers: this.headers, params: params }` 的参数形式。

```ts
{
    body?: any;
    headers?: HttpHeaders | {
        [header: string]: string | string[];
    };
    observe?: HttpObserve;
    params?: HttpParams | {
        [param: string]: string | string[];
    };
    reportProgress?: boolean;
    responseType?: 'arraybuffer' | 'blob' | 'json' | 'text';
    withCredentials?: boolean;
}
```

