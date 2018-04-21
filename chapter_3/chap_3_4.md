# 容器化我们的后端

就像我们在 `Angular` 中使用镜像一样，我们希望可以容器化我们的后端应用。

## 手动创建镜像

我们首先在 `backend/api/src/main` 下建立 `docker` 文件夹，然后在 `docker` 文件夹中创建一个 `Dockerfile`

![文件结构](/assets/2018-04-03-14-40-48.png)

```Dockerfile
// 以 java 8 jdk 镜像为基础
FROM openjdk:8-jdk-alpine
VOLUME ["/tmp"]
// 将编译后的 jar 文件加入容器
ADD api-0.0.1.jar app.jar
// 入口执行的方法
ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
// 镜像维护者的信息
MAINTAINER Peng Wang "wpcfan@gmail.com"
```

简单说明一下，既然是一个 JAVA 工程，我们就以 jdk 的镜像为基础来构建我们自己的镜像，要做的事情也很简单，就是把打包好的 jar 文件加入镜像。`ENTRYPOINT` 起的作用其实是相当于把数组中的各个元素拼成一个命令执行

```bash
java -Djava.security.egd=file:/dev/./urandom -jar /app.jar
```

我们可以测试一下这个 `Dockerfile` 是否可以成功创建镜像。在 `Dockerfile` 所在目录执行

```bash
docker build -t gtm/api:0.0.1 .
```

不出意外的话，你会遇到一个错误

```bash
Sending build context to Docker daemon  2.048kB
Step 1/5 : FROM openjdk:8-jdk-alpine
 ---> 224765a6bdbe
Step 2/5 : VOLUME ["/tmp"]
 ---> Using cache
 ---> 60efb1149bd3
Step 3/5 : ADD api-0.0.1.jar app.jar
ADD failed: stat /var/lib/docker/tmp/docker-builder039755918/api-0.0.1.jar: no such file or directory
```

这是由于 2 个原因，首先我们没有进行 `jar` 打包，其次 `Dockerfile` 中的 ADD 默认找的路径就是 `Dockerfile` 所在路径，而这个路径下没有这个 `jar`。找不到这个 `jar`，因此镜像创建失败了。知道了原因，我们就进行一下编译，在项目 `backend` 目录下执行

```bash
./gradlew :api:bootJar
cp ./api/build/libs/api-0.0.1.jar ./api/src/main/docker
```

然后再回到 `Dockerfile` 所在目录执行

```bash
docker build -t gtm/api:0.0.1 .
```

这次我们应该可以看到镜像创建成功了，但是 docker 目录里面的 jar 其实没啥用了，删掉它。

```log
Sending build context to Docker daemon  21.54MB
Step 1/5 : FROM openjdk:8-jdk-alpine
 ---> 224765a6bdbe
Step 2/5 : VOLUME ["/tmp"]
 ---> Using cache
 ---> 60efb1149bd3
Step 3/5 : ADD api-0.0.1.jar app.jar
 ---> c5cfd21a8410
Step 4/5 : ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
 ---> Running in 10a4837b2cf7
Removing intermediate container 10a4837b2cf7
 ---> a7853ee194f0
Step 5/5 : MAINTAINER Peng Wang "wpcfan@gmail.com"
 ---> Running in c3b210b305d2
Removing intermediate container c3b210b305d2
 ---> a5bd57615fd4
Successfully built a5bd57615fd4
Successfully tagged gtm/api:0.0.1
```

但这么操作也太麻烦了，嗯，我也这么觉得。所以我们要把这个过程自动化。

## 使用 Gradle 自动化 Docker 任务

使用 Gradle 或 Maven 这种脚本的好处之一在于你拥有了无限扩展的可能性，有 N 多的插件可以扩充这个脚本的能力。这里我们要使用的是一个 Gradle Docker plugin <https://github.com/bmuschko/gradle-docker-plugin> 。这个插件是专门给 gradle 脚本添加了很多 docker 的处理能力。这里我们只使用了几个：创建 Dockerfile，拷贝文件到镜像构建文件夹和创建镜像。

首先我们在 `backend` 下的 `gradle` 子目录中新建一个 `docker.gradle`

```groovy
buildscript {
    repositories {
        jcenter()
    }

    dependencies {
        classpath 'com.bmuschko:gradle-docker-plugin:3.2.6'
    }
}

apply plugin: com.bmuschko.gradle.docker.DockerRemoteApiPlugin

task createDockerfile(type: com.bmuschko.gradle.docker.tasks.image.Dockerfile, dependsOn: ['bootJar']) {
    description = "自动创建 Dockerfile"
    destFile = project.file('src/main/docker/Dockerfile')
    from 'openjdk:8-jdk-alpine'
    volume '/tmp'
    addFile "${project.name}-${project.version}.jar", "app.jar"
    instruction { 'ENTRYPOINT [' +
            '"java", ' +
            '"-Djava.security.egd=file:/dev/./urandom", ' +
            '"-jar","/app.jar"]'}
    maintainer 'Peng Wang "wpcfan@gmail.com"'
}

task copyDockerFiles(type: Copy, dependsOn: 'createDockerfile') {
    description = "拷贝 Dockerfile 和相应数据到镜像构建文件夹"
    from 'src/main/docker'
    from "${project.buildDir}/libs"
    into { "${project.buildDir}/docker" }
    include "*"
    exclude "**/*.yml"
}

task buildDocker(type: com.bmuschko.gradle.docker.tasks.image.DockerBuildImage, dependsOn: 'copyDockerFiles') {
    description = "打包应用构建镜像"
    group = "twigcodes"
    inputDir = project.file("${project.buildDir}/docker")
    tags = ["${project.name}:latest".toString(), "${project.name}:${project.version}".toString()]
}
```

这个文件是专门处理 docker 相关任务的，所以我们单独建立一个文件，以后其他类似情况我们也会为专门的任务建立 gradle 脚本文件，这样有助于我们管理一个日趋庞大的项目。

由于这个插件做的工作大部分是将我们之前的手动完成的动作自动化，大家可以参照官方文档和我们的代码自行体会一下，我这里就不展开讲了。

但有了这个文件，我们还需要在主文件中处理，所以在 `backend/build.gradle` 中的 `subprojects` 区块添加一行 `apply from: '../gradle/docker.gradle'` ，相当于为每个子项目添加了 docker 任务。

```groovy
// 略去其他部分
subprojects {
    // 略去
    apply plugin: 'java'
    apply plugin: 'idea'
    apply from: '../gradle/docker.gradle'
    apply plugin: 'eclipse'
    apply plugin: 'io.spring.dependency-management'
    // 略去
}
```

现在我们的任务可以完全自动化了，我们可以自动生成 `Dockerfile`

```bash
./gradlew :api:createDockerfile
```

可以自动化生成镜像

```bash
./gradlew :api:buildDocker
```

由于 `buildDocker` 依赖 `copyDockerFiles`，而 `copyDockerFiles` 依赖 `createDockerfile`，所以其实我们只需要执行 `buildDocker`，就会先生成 `Dockerfile`，然后创建镜像。这一个命令就全解决了。

镜像既然建立了，接下来，我们创建一个容器，来测试一下

```bash
docker run --rm --name api api:0.0.1
```

我们会发现出现了一些异常，如果使用 Postman 进行 API 测试也会返回错误。

```log
2018-04-03 15:03:53.912  INFO 1 --- [localhost:27017] org.mongodb.driver.cluster               : Exception in monitor thread while connecting to server localhost:27017

com.mongodb.MongoSocketOpenException: Exception opening socket
    at com.mongodb.connection.SocketStream.open(SocketStream.java:62) ~[mongodb-driver-core-3.6.3.jar!/:na]
    at com.mongodb.connection.InternalStreamConnection.open(InternalStreamConnection.java:126) ~[mongodb-driver-core-3.6.3.jar!/:na]
    at com.mongodb.connection.DefaultServerMonitor$ServerMonitorRunnable.run(DefaultServerMonitor.java:114) ~[mongodb-driver-core-3.6.3.jar!/:na]
    at java.lang.Thread.run(Thread.java:748) [na:1.8.0_151]
Caused by: java.net.ConnectException: Connection refused (Connection refused)
    at java.net.PlainSocketImpl.socketConnect(Native Method) ~[na:1.8.0_151]
    at java.net.AbstractPlainSocketImpl.doConnect(AbstractPlainSocketImpl.java:350) ~[na:1.8.0_151]
    at java.net.AbstractPlainSocketImpl.connectToAddress(AbstractPlainSocketImpl.java:206) ~[na:1.8.0_151]
    at java.net.AbstractPlainSocketImpl.connect(AbstractPlainSocketImpl.java:188) ~[na:1.8.0_151]
    at java.net.SocksSocketImpl.connect(SocksSocketImpl.java:392) ~[na:1.8.0_151]
    at java.net.Socket.connect(Socket.java:589) ~[na:1.8.0_151]
    at com.mongodb.connection.SocketStreamHelper.initialize(SocketStreamHelper.java:59) ~[mongodb-driver-core-3.6.3.jar!/:na]
    at com.mongodb.connection.SocketStream.open(SocketStream.java:57) ~[mongodb-driver-core-3.6.3.jar!/:na]
    ... 3 common frames omitted
```

这是由于我们将 API 容器化之后，它无法访问到 MongoDB 了，这就有点奇怪了，我们的 MongoDB 一直运行着啊，一直都是在 27017 端口，怎么容器化之后就访问不到了呢？可以这样理解，容器化相当于把服务关进一个小屋，服务可以看到这个小屋中的各种号码牌（把端口想像成号码牌），但是它看不到屋外的号码牌，MongoDB 在屋外立了一个 27017 的号码牌，但是在小黑屋里看不到啊。

这个问题怎么解决呢？要说简单也是非常简单，数据库的连接一般都是通过某种协议访问的，而 MongoDB 也不例外，它可以通过 TCP 连接： `mongodb://[IP/HOSTNAME]: [端口号]/[数据库名]` 。那么我们可以更改 docker 任务，传递这个数据库连接信息到 Spring Boot。

```groovy
task createDockerfile(type: com.bmuschko.gradle.docker.tasks.image.Dockerfile) {
    description = "自动创建 Dockerfile"
    destFile = project.file('src/main/docker/Dockerfile')
    from 'openjdk:8-jdk-alpine'
    volume '/tmp'
    addFile "${project.name}-${project.version}.jar", "app.jar"
    instruction { 'ENTRYPOINT ["java", "-Dspring.data.mongodb.uri=mongodb://mymongo/taskmgr", "-Djava.security.egd=file:/dev/./urandom", "-jar","/app.jar"]'}
    maintainer 'Peng Wang "wpcfan@gmail.com"'
}
```

增加的这行 `-Dspring.data.mongodb.uri=mongodb://mongo/taskmgr` 是会设置 `spring.data.mongodb.uri` 为 `mongodb://mymongo/taskmgr` 的。注意这里我们采用了主机名的形式，这是因为我们会在一台宿主机器上运行，并没有多个 `IP` 。看到这里，你可能会问那主机名不也是有问题吗，怎么让小黑屋里知道外部的主机名呢？ `docker` 提供了一个 `link` 参数，在启动容器时使用这个参数让容器可以和其他容器互相通信。

```bash
docker run --rm --name api api:0.0.1 --link mymongo
```

当然如果服务多起来之后，我们这样处理还是很麻烦的，所以就引出了我们对于多个服务的容器脚本 `docker-compose`

## 使用 docker-compose 组合服务

使用 `docker-compose` 的几个明显的好处是

1. 避免每次写带有 N 多参数的命令，脚本化之后效率肯定高啊
2. 需要启动多个服务，而且彼此有依赖的时候，这个构建过程会非常复杂，脚本可以帮我们清晰的定义关系，简化流程。

使用 `docker-compose` 需要在 `backend` 目录下建立一个 `docker-compose.yml`。在 `services` 区块下定义各项服务以及服务的各项参数。

```yml
version: '3.2'
services:
  mongo:
    image: mongo:3.4.10
    ports:
      - "27017:27017"
    volumes:
      - api_db:/data/db
  api-server:
    image: api
    ports:
      - "8080:8080"
    links:
      - mongo
volumes:
  api_db:
```

这个 `yml` 中的内容都比较浅显，我就不做详细解释了，具体 `docker-compose` 的用法可以参照官网文档 <https://docs.docker.com/compose/> 进行学习。

需要注意的一个地方是 `volumes:` 的作用，这个是做容器的文件夹和宿主的文件夹映射，主要作用有共享数据和保存数据两个方面。就数据库而言，一般建议将数据库的数据文件夹映射到宿主机器，否则容器销毁时，数据就全没了。

此外注意的一点是， docker-compose 中的各服务会单独创建容器，不会使用系统中已存在的容器。所以我们需要改写一下 `createDockerfile`，将 `mongodb://mymongo/taskmgr` 改成 `mongodb://mongo/taskmgr`，也就是把 `docker-compose.yml` 中的 MongoDB 的服务名 `mongo`

```groovy
task createDockerfile(type: com.bmuschko.gradle.docker.tasks.image.Dockerfile) {
    description = "自动创建 Dockerfile"
    destFile = project.file('src/main/docker/Dockerfile')
    from 'openjdk:8-jdk-alpine'
    volume '/tmp'
    addFile "${project.name}-${project.version}.jar", "app.jar"
    instruction { 'ENTRYPOINT ["java", "-Dspring.data.mongodb.uri=mongodb://mongo/taskmgr", "-Djava.security.egd=file:/dev/./urandom", "-jar","/app.jar"]'}
    maintainer 'Peng Wang "wpcfan@gmail.com"'
}
```

`docker-compose` 可以启动一组服务，在 `docker-compose.yml` 所在的目录执行下面命令，可以启动所有在 `yml` 定义的服务。

```bash
docker-compose up -d
```

结束这一组服务，也很简单

```bash
docker-compose down
```

## IDE 中的 Gradle 支持

在 IDEA 中，我们可以通过 View -> Tool Window -> Gradle 调出 Gradle 工具窗口。

![调出 IDEA 中的 Gradle 工具窗口](/assets/2018-04-03-16-19-25.png)

在工具窗口可以双击某个具体任务执行

![在 Gradle 中可以双击执行某个任务](/assets/2018-04-03-16-21-55.png)

## 在容器中调试

形成容器之后，部署是比较方便，那可能有同学要问，调试怎么破呢？其实这个也很简单，你知道容器其实就是一个类似虚拟机的存在，那么主流 IDE 对于远程调试都有良好的支持，我们可以把容器看作一个远程主机，所以调试当然也不成问题。

我们唯一要做的是在创建 `Dockerfile` 时，增加 `ENTRYPOINT` 的一个参数 `-agentlib:jdwp=transport=dt_socket,address=5005,server=y,suspend=n` ，这个其实是为了远程调试而设置数据传输的协议和监听端口等。现在我们的 `createDockerfile` 看起来是下面的样子了。

```groovy
task createDockerfile(type: com.bmuschko.gradle.docker.tasks.image.Dockerfile) {
    description = "自动创建 Dockerfile"
    destFile = project.file('src/main/docker/Dockerfile')
    from 'openjdk:8-jdk-alpine'
    volume '/tmp'
    addFile "${project.name}-${project.version}.jar", "app.jar"
    instruction { 'ENTRYPOINT ["java", "-agentlib:jdwp=transport=dt_socket,address=5005,server=y,suspend=n", "-Dspring.data.mongodb.uri=mongodb://mongo/taskmgr", "-Djava.security.egd=file:/dev/./urandom", "-jar","/app.jar"]'}
    maintainer 'Peng Wang "wpcfan@gmail.com"'
}
```

设置好这些，我们重新生成镜像

```bash
./gradlew :api:buildDocker
```

由于在容器内多了一个监听远程调试的端口，我们也需要在 `docker-compose.yml` 中多加一个 `5005:5005` 的端口映射

```yml
version: '3.2'
services:
  mongo:
    # 略过
  api-server:
    image: api
    ports:
      - "8080:8080"
      - "5005:5005"
    links:
      - mongo
# 略过
```

我们就可以在 IDEA 中 `RUN -> Edit Configurations` 中新建一个 `Remote` 类型的 `Run/Debug Configuration`。

![新建 Remote 的 Run/Debug 配置](/assets/2018-04-04-00-25-03.png)

此时如果你还没有启动 `docker-compose` 的话，请执行 `docker-compose up -d` 。然后启动 IDE 中选中我们刚刚建立的 Remote 的配置，点击调试。如果你看到下面这句话，那么我们的配置就全 OK 了。

```log
Connected to the target VM, address: 'localhost:5005', transport: 'socket'
```

![调试连接建立成功](/assets/2018-04-04-00-50-38.png)

