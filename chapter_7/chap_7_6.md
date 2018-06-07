# WebSocket 实时通讯服务

WebSocket 协议（ RFC 6455 ）提供了一种标准化的方式来进行服务器和客户端之间进行双工、双向的基于 TCP 连接的通讯方式。它既不同于 HTTP ，又构建于 HTTP 之上，它可以使用 HTTP 的协议端口，比如 `80` 或 `443` 。

一个 WebSocket 交互是由一个 HTTP 请求开始的，这个 HTTP 请求使用 "Upgrade" 头去转换到 WebSocket 协议：

```txt
GET ws://localhost:8080/websocket/app HTTP/1.1
Host: localhost:8080
Connection: Upgrade
Pragma: no-cache
Cache-Control: no-cache
Upgrade: websocket
Origin: chrome-extension://heeinifidncnmblmddfajlikkiihfgai
Sec-WebSocket-Version: 13
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_4) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/66.0.3359.181 Safari/537.36
Accept-Encoding: gzip, deflate, br
Accept-Language: zh-CN,zh;q=0.9,en-US;q=0.8,en;q=0.7
Cookie: _ga=GA1.1.13036277.1526997915; m=
Sec-WebSocket-Key: I88buTTIKcJE1QKxHAMpnA==
Sec-WebSocket-Extensions: permessage-deflate; client_max_window_bits
Sec-WebSocket-Protocol: v10.stomp, v11.stomp, v12.stomp
```

支持 WebSocket 的服务器端收到这个请求后，会返回

```txt
HTTP/1.1 101 Switching Protocols
Expires: 0
Cache-Control: no-cache, no-store, max-age=0, must-revalidate
X-XSS-Protection: 1; mode=block
Origin: chrome-extension://heeinifidncnmblmddfajlikkiihfgai
Upgrade: WebSocket
Pragma: no-cache
Sec-WebSocket-Accept: G+NHta8H3Mjiol8xeT3T3Tilabw=
Date: Thu, 07 Jun 2018 04:45:14 GMT
Connection: Upgrade
Sec-WebSocket-Location: ws://localhost:8080/websocket/app
X-Content-Type-Options: nosniff
Sec-WebSocket-Protocol: v10.stomp
```

在成功的握手之后，TCP socket 会保持对服务器和客户端打开以便发送和接收消息。注意如果 WebSocket 服务器在 Web 服务器后面，比如有时我们采用 Web 服务器是 nginx ，通过反向代理把某些请求指向我们的 API 服务，这种时候就需要配置 nginx 将 WebSocket 的 Upgrade 请求发送到 WebSocket 服务器。

## HTTP 和 WebSocket 的区别和联系

我们在考虑使用 WebSocket 的时候要注意，通常的 Rest API 模型和 WebSocket 模型的区别是巨大的，无论对服务端还是客户端来说，架构和编程模式上的改变都是如此。在 Rest API 模型中，我们把资源划分成若干 URL ，客户端需要使用 Request/Response 的机制去访问这些 URL ，服务端也会根据这些 Request 的 URL 、方法和请求头来决定对应的处理方式。

而对于 WebSocket 来说，通常是建立一个连接后就通过这个连接发送后继的消息，这个是一个一直保持连接的模型，而不是像 REST 方式那样每次建立新的连接。一般来说，这种模式要求的编程模型是事件驱动类型的，和 Request/Response 的模型区别较大。

另外 WebSocket 是一个较基础的传输协议，没有像 HTTP 那样规定内容的格式、语义，我们一般需要在其之上再封装一层协议来处理内容、格式等，比如我们下面要谈到的 STOMP。

## 何时使用 WebSocket？

初学者往往觉得既然可以实时，那就所有业务都实时化好了，但实时是有成本的，而且大多数场景不需要实时，对这些场景进行实时化处理代价高昂，却收益甚微。

WebSockets 主要用于处理实时性较高的需求，但它并不是这类问题的唯一解决方案，在很多时候，利用 Ajax 或者 HTTP 流或者 HTTP 长连接等方式可以获得类似的效果和体验，甚至很多场景下，用其他方案会比 WebSocket 更好。

并非所有应用都要求实时性，多人在线游戏和股票类应用需要更高的实时性，而新闻类，邮件类或者社交类应用往往只需要阶段性的更新即可。除了时延，数据的传输量也是一个考虑因素，如果数据量不大的情况下，使用 HTTP 长链接（Long Pooling）应该是更好的选择。此外网络环境也是一个考虑因素，如果我们的 Web 代理并不支持，或者使用的云服务不支持 Upgrade 到一个需要常驻的连接的话，那么我们也就无法使用 WebSocket 了。

## STOMP

和 `Node.js` 生态中的 `Socket.js` 类似， `Spring` 采用了一个叫做 `STOMP` 的子协议。 `STOMP` 是由 Google 开发的一套基于 WebSocket 之上的消息通讯协议，和传统的 AMQP 协议以及 JMS 的作用类似，但不像传统的协议那么复杂，只包含最常见的一些操作。
