# 容器化你的 Angular 应用

## 什么是容器？

容器其实可以理解成一个硬件虚拟化后的运行镜像，如果认为虚拟机是模拟运行的一套操作系统（提供了运行态环境和其他系统环境）和跑在上面的应用。那么 Docker 容器就是独立运行的一个或一组应用以及它们的运行环境。

![虚拟机和容器架构](/assets/2018-03-09-11-50-35.png)

图中左侧是虚拟机的架构，图中的例子有 4 个操作系统（Operating System）。他们是 1 个 Host Operating System 和 3 个 Guest Operating System，每个虚拟机中都有一个独立的 Kernel。虚拟机可以让你能充分利用你的硬件资源。你可以购买性能超强的物理机器，然后在上面运行多个虚拟机。你可以有一个数据库虚拟机以及很多运行相同版本的定制应用程序的虚拟机所构成的集群。你可以在有限的硬件资源获得很多的扩展能力。如果你觉得你需要更多的虚拟机而且你的宿主硬件还有容量，你可以添加任何你需要的虚拟机。如果你不再需要一个虚拟机，你可以关闭该虚拟机甚至删除虚拟机镜像。但虚拟机的缺陷在于所有分配给一个虚拟机的资源是专有的。假如一台物理主机有 8 GB 的内存，如果我们建立一个拥有 1G 内存的虚拟机，那么它只会使用这 1GB 内测，即使内存不足，它也不会动用剩余的 7 GB 内存。没有资源的动态分配，每台虚拟机**拥有且只能使用**分配给它的资源。

而图中右侧为容器架构，容器其实是一个进程，操作系统认为它只是一个运行中的进程。另外，该容器进程也分配了它自己的 IP 地址。一旦有了一个 IP 地址，该进程就是宿主网络中可识别的资源。然后，你可以在容器管理器上运行命令，使容器 IP 映射到主机中能访问公网的 IP 地址。建立了该映射，容器就是网络上一个可访问的独立机器，从概念上类似于虚拟机。

因为容器是一个进程，所以它可以动态的共享主机上的资源。如果容器只需要 1GB 内存，它就只会使用 1GB。如果它需要 4GB，就会使用 4GB。CPU 和硬盘等也是如此。和典型虚拟机的静态方式不同。所有这些资源的共享都由容器管理器来管理。而且，由于摆脱了硬件虚拟化，容器可以非常快速地启动。

容器的好处是：既有虚拟机独立和封装的优点，又屏蔽了静态资源专有的缺陷，可以共享资源。另外，由于容器能快速加载到内存，在扩展到多个容器时你能获得更好的性能。

那么对于开发者来说，现在由于容器既快速又方便，这就解决了部署的痛苦。有经验的开发者都知道开发环境好用的应用到了生产环境会出现各种问题，因为现代的软件依赖很多第三方类库和其他环境支持，稍微有点变化就会导致程序无法正确运行。 Docker 给开发者带来的好处就是可以让开发环境和生产环境一致，无缝部署到生产环境。

## 安装 Docker

一般来说在 Linux 主机上，可以使用官方的安装脚本，国内的各种云服务购买的 Linux 虚拟机上都可以这么安装。

```bash
curl -sSL https://get.docker.com/ | sh
```

在 Mac 或者 Windows 上面，需要到 https://www.docker.com/community-edition#/download 下载官方的安装包

![为 Windows 或 Mac 下载 Docker](/assets/2018-03-09-12-21-09.png)

## 镜像仓库加速

由于众所周知的原因，很多开源的开发工具或软件仓库都在国内访问速度欠佳，这时候一般采用的都是使用国内的镜像进行加速。Docker 也不例外，我们这里提供一种使用腾讯云提供的 Docker 软件镜像仓库，可以加速镜像下载。

在 Linux 系统中可以采用下面脚本进行设置

```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://mirror.ccs.tencentyun.com"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```

## 创建 Angular 的 Docker 镜像

将容器应用到我们的 Angular 项目中是比较简单的，首先在项目工程的根目录建立一个叫 `docker` 的文件夹，然后在 `docker` 文件夹下面建立 `nginx` 文件夹，并在此文件夹中创建一个叫 `Dockerfile` 的文件。

我们在这个 Dockerfile 中要做两件事：

1.  编译 Angular 应用，编译后的文件会存在 `dist` 目录下
2.  将 `dist` 目录下的 `html`，`css` 和 `javascript` 文件发布到 `nginx`

为了更好的理解后面的容器话过程，我们先手动完成这样的部署，先编译项目，在项目根目录执行下面的命令

```bash
ng build --prod
```

如果编译成功的话，你会看到类似下面这样的输出，如下图所示，成功编译后会产生几个 `html`，`css` 和 `javascript` 文件，这些文件位于项目根目录下的一个叫 `dist` 的子目录。

![编译结果](/assets/2018-03-09-15-09-51.png)

如果我们把这些文件拷贝到一个 HTTP 服务器的 web 目录，就可以访问我们生成的页面了。如果你的电脑上已经安装了 nginx 的话，你可以把 dist 目录下的内容拷贝到 `/usr/share/nginx/html` 目录下，然后看看效果。

其实 Docker 要做的工作和上述手动的过程类似，在编译过程中，我们分别需要 `node` 镜像，就相当于安装 node.js 环境，编译之后的发布其实只需要把内容拷贝到 `nginx` 镜像的 web 目录下。

```Dockerfile
# 第一阶段：构建 Angular 应用 ###

# 编译这个阶段需要的是 node 镜像，所以我们以 'node:9-alpine' 为基础
# 另外给这一阶段起个友好名称叫 builder，以便于后面第二阶段可以方便的引用第一阶段的成果
FROM node:9-alpine as builder

# 单独拷贝 'package.json'，在下一层安装相关依赖
COPY package.json ./

# 把 node_modules 保存在单独的一层以避免以后每次构建时都重复做 npm install
# 由于 npm 的访问速度实在感人，所以我们采用阿里团队提供的镜像
RUN npm config set registry https://registry.npm.taobao.org && npm i && mkdir /ng-app && cp -R ./node_modules ./ng-app

# 指定工作目录
WORKDIR /ng-app

COPY . .

# 在生产模式下编译 Angular 应用
ARG env=production
RUN npm run build -- --prod --configuration $env

# 第二阶段：设置 Nginx 服务器 ###

FROM nginx:1.13.8-alpine

# 将我们的 nginx 配置文件拷贝到镜像中的 /etc/nginx/conf.d/ 目录
COPY ./docker/nginx/conf.d/default.conf /etc/nginx/conf.d/

# 删除默认站点
RUN rm -rf /usr/share/nginx/html/*

# 从 'builder' 阶段将编译后的文件拷贝到 nginx 默认的站点目录 ‘/usr/share/nginx/html’
COPY --from=builder /ng-app/dist /usr/share/nginx/html

# 执行命令启动 Nginx
CMD ["nginx", "-g", "daemon off;"]
```

从上面的文件可以看出来，我们如果想构建一个镜像的话，首先可以先从现有的镜像为基础去更改。比如第一阶段中我们需要 node.js 环境去编译 Angular 应用，所以我们就使用 `node:9-alpine` 镜像，在 Dockerfile 中很多操作就是在把我们手动的环境配置写进去，比如 `COPY` 、 `RUN` 命令等。而且很棒的一点是 Dockerfile 还支持多阶段镜像，也就是说我们可以基于多个镜像，这个例子中，我们第一阶段使用了 node ，而第二个阶段使用了 nginx 。

那么接下来我们就可以用这个 Dockerfile 编译我们自己的镜像了，执行下面的命令生成镜像文件

```bash
# '-f' 开关指定使用的 Dockerfile 文件
# '-t' 开关指定容器的名字，也就是下面的 ng-app
# 'ng-app' 后面的 ‘.’ 是指定上下文路径，Dockerfile 中的 COPY 等命令会基于这个路径来操作
docker build -t ng-app . -f ./docker/nginx/Dockerfile
```

## 启动容器

启动这个容器再简单不过，做好端口映射， `-p <本地端口>:<容器端口>` 用于指定本地和容器的端口映射， 注意前面的端口是本地端口。如果没有 `SSL` 要求或者没有配置 `HTTPS` 的话， 其实是不用  加`-p 443:443` 的。命令中的 `--rm` 表示这是个临时容器，如果容器停止的话就会删除掉。命令中的 `-d` 表示在后台运行，而最后的 `gtm:dev` 就是我们上一步生成的镜像名称了。

```bash
docker run --rm -d -p 443:443 -p 80:80 gtm:dev
```

容器启动后，你可以访问 `http://localhost` 来看到我们目前的成果。

## 使用 docker-compose 组织复杂的环境配置

真正的开发环境或者生产环境  一般不是由单纯的某一个服务构成的， 而是由一组服务构成的，比如有负责前端渲染的服务器，有后端的 API 服务，有数据库服务等等。所以当我们构建了一个容器后面临的一个问题就是如何组织  项目依赖的各个服务。

 我们可以通过在工程中建立一个 `docker-compose.yml` 文件来  实现  上述目标。我们把这个文件建立在工程根目录即可。这个文件是 `yml` 格式的，可读性很强，简单来说就是 **以缩进体现结构，以键/值对体现属性设置** 。

 在 `services` 下面可以  列出项目需要的所有服务，目前我们只有一个 `nginx` ，所以你只能看到一个，但随着  后面的学习，会逐渐增加进去的。

```yml
version: '3'
services:
  nginx:
    build:
      context: .
      dockerfile: ./docker/nginx/Dockerfile
      args:
        - env=dev
    container_name: nginx
    ports:
      - 80:80
```

有了这个 `docker-compose.yml` 文件，我们怎么使用呢？

使用下面的命令进行整个 docker-compose 中需要的镜像的构建，并在构建后启动所有服务。一般在更新了  镜像所需的文件时，采用这个命令重新构建。

```bash
docker-compose up --build
```

上面的命令如果没有错误产生的话，输出应该是类似下面的样子：

```terminal
Building nginx
Step 1/12 : FROM node:8-alpine as builder
 ---> 7c2983dfbf98
Step 2/12 : COPY package.json ./
 ---> Using cache
 ---> bcb0d43e7df7
Step 3/12 : RUN npm i && mkdir /ng-app && cp -R ./node_modules ./ng-app
 ---> Running in 789733c11030
npm WARN deprecated nodemailer@2.7.2: All versions below 4.0.1 of Nodemailer are deprecated. See https://nodemailer.com/status/
npm WARN deprecated mailcomposer@4.0.1: This project is unmaintained
npm WARN deprecated socks@1.1.9: If using 2.x branch, please upgrade to at least 2.1.6 to avoid a serious bug with socket data flow and an import issue introduced in 2.1.0
npm WARN deprecated node-uuid@1.4.8: Use uuid module instead
npm WARN deprecated buildmail@4.0.1: This project is unmaintained
npm WARN deprecated socks@1.1.10: If using 2.x branch, please upgrade to at least 2.1.6 to avoid a serious bug with socket data flow and an import issue introduced in 2.1.0

> uws@9.14.0 install /node_modules/uws
> node-gyp rebuild > build_log.txt 2>&1 || exit 0


> node-sass@4.8.3 install /node_modules/node-sass
> node scripts/install.js

Downloading binary from https://github.com/sass/node-sass/releases/download/v4.8.3/linux_musl-x64-57_binding.node
Download complete
Binary saved to /node_modules/node-sass/vendor/linux_musl-x64-57/binding.node
Caching binary to /root/.npm/node-sass/4.8.3/linux_musl-x64-57_binding.node

> uglifyjs-webpack-plugin@0.4.6 postinstall /node_modules/webpack/node_modules/uglifyjs-webpack-plugin
> node lib/post_install.js


> node-sass@4.8.3 postinstall /node_modules/node-sass
> node scripts/build.js

Binary found at /node_modules/node-sass/vendor/linux_musl-x64-57/binding.node
Testing binary
Binary is fine
npm notice created a lockfile as package-lock.json. You should commit this file.
npm WARN ajv-keywords@3.1.0 requires a peer of ajv@^6.0.0 but none is installed. You must install peer dependencies yourself.
npm WARN optional SKIPPING OPTIONAL DEPENDENCY: fsevents@1.1.3 (node_modules/fsevents):
npm WARN notsup SKIPPING OPTIONAL DEPENDENCY: Unsupported platform for fsevents@1.1.3: wanted {"os":"darwin","arch":"any"} (current: {"os":"linux","arch":"x64"})

added 1558 packages in 233.974s
Removing intermediate container 789733c11030
 ---> 54a94bef793a
Step 4/12 : WORKDIR /ng-app
Removing intermediate container 6f9b336dc0e8
 ---> 9d6e2a9c8deb
Step 5/12 : COPY . .
 ---> 1b9cc29a0831
Step 6/12 : ARG env=prod
 ---> Running in 0eb37fa6530c
Removing intermediate container 0eb37fa6530c
 ---> 2234f5373e66
Step 7/12 : RUN npm run build -- --prod --configuration $env
 ---> Running in 3d26db2ca897

> gtm@0.0.0 build /ng-app
> ng build "--prod" "--configuration" "dev"

Date: 2018-03-31T14:09:08.208Z
Hash: 2e6fc11c769df2c5d8e8
Time: 51145ms
chunk {0} main.d46b2e467d69bc70f0dd.bundle.js (main) 915 kB [initial] [rendered]
chunk {1} polyfills.cc16b94dcfd6baec7194.bundle.js (polyfills) 60.1 kB [initial] [rendered]
chunk {2} styles.77df39064a08875d96f3.bundle.css (styles) 89.3 kB [initial] [rendered]
chunk {3} inline.8460b3901605c8e449fe.bundle.js (inline) 1.45 kB [entry] [rendered]
Removing intermediate container 3d26db2ca897
 ---> dfe339b68abf
Step 8/12 : FROM nginx:1.13.8-alpine
 ---> bb00c21b4edf
 ---> bb00c21b4edf
Step 9/12 : COPY ./docker/nginx/conf.d/default.conf /etc/nginx/conf.d/
 ---> adf15ab968f4
Step 10/12 : RUN rm -rf /usr/share/nginx/html/*
 ---> Running in c6a67abeb18c
Removing intermediate container c6a67abeb18c
 ---> 2cb90aad97a1
Step 11/12 : COPY --from=builder /ng-app/dist /usr/share/nginx/html
 ---> 4f2d2de05dc1
Step 12/12 : CMD ["nginx", "-g", "daemon off;"]
 ---> Running in 437cb9c00d5b
Removing intermediate container 437cb9c00d5b
 ---> 522c49bf937c
Successfully built 522c49bf937c
Successfully tagged gtm_nginx:latest
Creating nginx ...
Creating nginx ... done
Attaching to nginx
```

 此时，可以访问 `http://localhost` 来看看我们的成果。

启动全部  服务，可以采用下面的命令，其中 `-d` 表示后台运行。

```bash
docker-compose up -d
```

停止全部  服务，可以采用下面的命令：

```bash
docker-compose down
```

 重新启动 `docker-compose` 中的某一项服务：

```bash
docker-compose restart nginx
```

查看某一项服务的日志

```bash
docker-compose logs nginx
```

## 使用 .dockerignore 文件

和 git 很像的一点是，容器中也有类似 `.gitignore` 的文件，叫做 `.dockerignore` 。它们起的作用  也非常相似，都是要过滤掉一些文件 ，也就是在这个文件中列出的文件都不会参与到容器的构建动作中，对于容器来说，这些文件是**不可见的**。

所以务必要注意，如果你在 `Dockerfile` 中  写了类似 `COPY` 这样的命令的话，一定记得要检查一下是否有你想使用的目录 ，却在 `.dockerignore` 文件中列出了。

```bash
# dependencies
node_modules/
# compiled output
dist/
dist-server/
tmp/
out-tsc/
.git/
.vscode/
```
