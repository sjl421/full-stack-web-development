# 打造后端 API

## 权限的设计

### ACL 权限模型

`ACL` 是英文 `Access Control Lists` 的缩写， `ACL` 指的是对于某个数据对象的权限列表，一个 `ACL` 会指出哪些用户或系统进程被授予了对数据对象的访问权限，以及允许什么样的操作。比如文件的 `ACL` 通常是类似 `(ZhangSan: read, write; LiSi: read)` 。

### 基于 RBAC 的权限模型

`RBAC` 是 `Role-Based Access Control` 的英文缩写，翻译过来就是基于角色的访问控制。`RBAC` 认为权限授权实际上是 `Who` 、 `What` 、 `How` 决定的。在 `RBAC` 模型中，`Who` 、 `What` 、 `How` 构成了访问权限三元组，即 `Who` 对 `What` 进行 `How` 的操作。其中 `Who` 是权限的拥有者或主体（如：`User` 、 `Role` ）， `What` 是资源或对象（`Resource` 、 `Class`)

`RBAC` 主要分为四种变化形式：

* 核心模型 `RBAC-0`（ `Core RBAC` ）
* 角色分层模型 `RBAC-1`（ `Hierarchal RBAC` ）
* 角色限制模型 `RBAC-2`（ `Constraint RBAC` ）
* 统一模型 `RBAC-3`（ `Combines RBAC` ）

最重要也是最基本的是 `RBAC-0`，因为这个模型是最小化实现 `RBAC` 权限思想的方式，其他的都是在此基础上的补充和变化。

![RABC 领域模型](/assets/2018-04-07-18-07-32.png)

`RBAC` 和 `ACL` 的区别在于 `RBAC` 将权限分配到对组织有意义的特定操作上而不是分配到底层的数据对象上。举个小例子，ACL 可以用来授予或拒绝某个系统文件的写访问请求，但它不能判断这个文件是怎样被更改的。在 `RBAC` 系统中，一个操作可以是在一个财务系统中“创建一个账簿”或者在一个医疗系统中“执行一个血糖测试”。这些操作的权限分配对组织来讲是有意义的，因为组织内的这些操作是一个基本单位的流程。 `RBAC` 非常适合职责分离的需求，这种需求下经常会要求确保至少 2 个或 2 个以上的人员参与到授权的操作中去。

其实一个最小化的 RBAC 模型和 `ACLg` （带分组的 `ACL` ）是等效的。

### 领域对象

从前端来看，有权限要求的、功能上比较简单的登录注册需要的领域对象有两个：用户（ `User` ）和角色（ `Role` ）。

```ts
import { Role } from './role';

export interface Role {
  name: string;
  permissions: string[];
}

export interface User {
  id: string;
  name: string;
  password?: string;
  username: string;
  mobile: string;
  email: string | null;
  roles: Role[];
}
```
