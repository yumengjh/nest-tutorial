---
title: 单例模式和请求级作用域
createTime: 2025/06/23 10:42:00
permalink: /nest/ylhwjrd4/
---


## 注入作用域{#injection-scopes}

对于来自不同编程语言背景的开发者来说，Nest 中几乎所有内容在请求之间都是共享的，这可能让人意外。比如，我们通常使用数据库连接池或带有全局状态的单例服务。请记住，Node.js 不同于传统多线程无状态请求/响应模型，在 Node.js 中，所有请求由单线程事件循环处理，因此使用单例实例在大多数情况下是安全且高效的。

不过，在某些特殊场景下，基于请求生命周期的服务行为是必须的，例如 GraphQL 应用中的每请求缓存、请求追踪以及多租户支持。注入作用域机制允许我们为提供者指定不同的生命周期，从而满足这些需求。

## 提供器作用域{#provider-scope}

NestJS 中的提供者 (Provider) 是通过依赖注入机制被应用系统管理的类实例。

Nest 提供 **3 种作用域 (Scope)** ：

| Scope 类型         | 生命周期       | 是否共享           | 适用场景           |
| ------------------ | -------------- | ------------------ | ------------------ |
| `DEFAULT` (单例)   | 应用启动时     | 是，全应用共享     | 配置服务，DB连接池 |
| `REQUEST` (请求级) | 每个请求开始时 | 不，各请求独立存在 | 多租户，访问追踪   |
| `TRANSIENT` (瞬态) | 每次注入时     | 不，每次新建       | 简单静态计算       |

- **DEFAULT** ：提供程序的单个实例在整个应用中共享。实例生命周期与应用生命周期直接相关。应用启动后，所有单例提供程序都已实例化。默认情况下使用单例作用域。
- **REQUEST** ：专门为每个传入的**请求**创建一个新的提供程序实例。请求完成处理后，该实例将被垃圾回收。
- **TRANSIENT** ： 临时提供器不在消费者之间共享。每个注入临时提供器的消费者都将收到一个新的专用实例。

对于大多数用例，建议使用单例范围。跨消费者和跨请求共享提供器意味着一个实例可以被缓存并且它的初始化只发生一次，在应用启动期间。

**各作用域解析**

**DEFAULT （单例）**

```ts
@Injectable() // 等同于 @Injectable({ scope: Scope.DEFAULT })
export class ConfigService {
  private readonly options = loadConfig();
  get(key: string) {
    return this.options[key];
  }
}
```

**特性：**

- Nest 应用启动时初始化一次，全应用共享
- 适合无状态服务：日志、配置、DB 连接池
- 性能最优，内存占用最低

**风险：**

- 不适合保存请求级状态（如 currentUser）
- 如果不小心使用全局变量保存状态，容易造成跨请求数据乱用

**REQUEST （请求级）**

```ts
@Injectable({ scope: Scope.REQUEST })
export class UserContextService {
  private userId: string;
  setUser(id: string) {
    this.userId = id;
  }
  getUserId() {
    return this.userId;
  }
}
```

**特性：**

- Nest 会为每个请求单独创建实例
- 适用于需要独立保存请求信息的场景
- 支持 GraphQL的 dataloader 隔离

**缺点：**

- 性能有所损耗（每次请求都会创建新实例）
- ~~不能被单例服务注入~~，~~否则会报错~~（会因为冒泡机制把单例也变为请求级）

**TRANSIENT （瞬态）**

```ts
@Injectable({ scope: Scope.TRANSIENT })
export class CalculatorService {
  getTimestamp() {
    return Date.now();
  }
}
```

**特性：**

- 注入谁，创建新实例，不保存
- 适合需要定制独立计算的场景
- 少量场景使用，性能费用最高

**实际场景指南**

| 场景               | 推荐 Scope | 说明                               |
| ------------------ | ---------- | ---------------------------------- |
| 配置服务           | DEFAULT    | 无状态，启动初始化一次             |
| 日志服务           | DEFAULT    | 无状态，日志打印都共用一个实例     |
| 用户上下文         | REQUEST    | 需要请求隔离，各自存储 currentUser |
| GraphQL dataloader | REQUEST    | 各请求独立 cache                   |
| 简单事务计算       | TRANSIENT  | 每次注入都要新建实例               |

**最佳实践**

1. 默认使用 **DEFAULT** (单例)，最简单高效
2. 有明确隔离需求时，使用 **REQUEST**
3. ~~禁止单例服务注入 request 级服务，NestJS 会报错~~
4. 若需传递 request 上下文，使用 `@Inject(REQUEST)`

**行为对比**

| 维度      | DEFAULT (单例) | REQUEST (请求级) | TRANSIENT (瞬态) |
| --------- | -------------- | ---------------- | ---------------- |
| 性能      | 高             | 中               | 最低             |
| 内存      | 最低           | 中               | 最高             |
| 缓存/保存 | 是             | 是（同请求内）   | 否               |
| 隔离性    | 否             | 是               | 是               |
| 适用场景  | DB连接池/配置  | 用户上下文/trace | 注入时加工       |

能用单例，尽量用单例，有需求隔离时，用 REQUEST ，较简单而静态，瞬态并非常用，主要用于特殊场景，不推荐普遍使用。

**同一个请求内创建的所有 `REQUEST` 作用域服务实例是共享的**。

Nest 会为每个传入请求创建一个 **独立的“请求上下文容器”**，该容器内的 `REQUEST` 作用域服务实例会被缓存。如果多个服务或控制器在同一个请求中依赖同一个 `REQUEST` 作用域的服务，它们会接收到 **相同的实例**，从而实现数据共享。

**示例**

```ts
@Injectable({ scope: Scope.REQUEST })
export class RequestContextService {
  private requestId = uuid();
  getRequestId() {
    return this.requestId;
  }
}
```

```ts
@Controller()
export class SomeController {
  constructor(private readonly ctx: RequestContextService) {}

  @Get()
  handle() {
    return this.ctx.getRequestId(); // 每次请求生成的新 ID
  }
}
```

```ts
@Injectable()
export class AnotherService {
  constructor(private readonly ctx: RequestContextService) {}

  logRequest() {
    console.log(this.ctx.getRequestId()); // 同一个请求中和 Controller 得到的是同一个 ID
  }
}
```

只要这两个类都是处理 **同一请求** 的，它们拿到的是 **同一个 RequestContextService 实例**。

注意：

**不同请求之间的数据不会共享**

单例作用域（`DEFAULT`）服务不能依赖注入 `REQUEST` 服务，~~否则 Nest 会报错~~（因为生命周期不兼容，会因为冒泡机制把单例也变为请求级）

```ts
// ❌ 错误用法：默认作用域（单例）注入请求级作用域
@Injectable()
export class SingletonService {
  constructor(private readonly ctx: RequestContextService) {}
}

// Error: Nest can't resolve dependencies of the SingletonService (?). Please make sure that the argument at index [0] is available...
```

正确做法：将 `AnotherService` 也声明为 `REQUEST` 作用域

若需要在单例中访问请求信息，应使用 `@Inject(REQUEST)` 注入 Express 的 Request 对象，但：
- **该方法仅限访问原始请求数据**，如 headers、query、params、body、IP、user 等（即从 HTTP 请求中解析的基础信息）
- **不能通过这种方式访问 Nest 管理的请求级服务实例**，例如 `UserContextService`。这是因为这些服务是由 Nest 基于请求上下文容器动态创建和管理的，只有声明为 `REQUEST` 作用域的类中才能通过构造函数注入得到它们。
- 简单理解：你可以拿到“请求数据”，但拿不到“跟这个请求绑定的 Nest 服务”

**总结**

- 🚫 **单例服务永远不应该“知道”请求级状态**
- ✅ **请求状态应由控制器或请求服务掌握并向下传**
- 🧠 **职责清晰、状态隔离是框架设计的底线**
- ⚠️ Nest 强制禁止的，其实就是你业务逻辑上本不该做的

**实际场景**：如果某个单例服务内部需要用到当前请求级数据（如当前用户 ID），该怎么办？

错误方式：

```ts
@Injectable()
export class SingletonService {
    constructor(private readonly userCtx: UserContextService) {} // ❌ 直接注入请求级服务，Nest 启动时报错
}
```

正确做法：**由调用者在同一个请求中，将数据“向下传”给单例服务**

也就是说，请求级数据不要让服务自己“去找”，而是由请求上下文**的上层代码传下去**（类似函数参数的方式）

示例：

```ts
// ✅ 单例服务，不知道请求，也不依赖请求服务
@Injectable()
export class LoggerService {
  log(message: string, traceId: string) {
    console.log(`[${traceId}] ${message}`);
  }
}
```

```ts
// ✅ 请求级服务：知道当前 traceId
@Injectable({ scope: Scope.REQUEST })
export class RequestContextService {
    private traceId: string;
    setTraceId(id: string) {
        this.traceId = id;
    }
    getTraceId() {
        return this.traceId;
    }
}
```

```ts
// ✅ 控制器中协调：读取请求级数据，传给单例服务
@Controller()
export class AppController {
    constructor(
    private readonly ctx: RequestContextService,
     private readonly logger: LoggerService
    ) {}

    @Get()
    handle() {
        const traceId = this.ctx.getTraceId();
        this.logger.log('处理完成', traceId);
        return 'OK';
    }
}
```

| 错误方式                                 | 正确方式                         |
| ---------------------------------------- | -------------------------------- |
| ❌ 服务自己注入另一个生命周期不一致的服务 | ✅ 上层读取后传参                 |
| ❌ 单例服务想“偷”请求状态                 | ✅ 请求级服务调用单例时传递状态值 |

**“在一个请求中同时使用单例和请求级服务” 是允许的、也是常态。**
 关键是：**单例不要去“注入”请求级服务，而是由请求路径控制谁调用谁、谁传值给谁。**

**实战建议**

| 学到的知识       | 实战中该怎么用                                               |
| ---------------- | ------------------------------------------------------------ |
| 请求级作用域     | 理解原理即可，能不用就不用，参数传递更清晰                   |
| 依赖注入         | 习惯了就好，不用纠结底层实现，关注结构划分即可               |
| 装饰器元编程     | 有用时才写，过度封装会降低团队可读性                         |
| 生命周期与 scope | 在做大型系统时确实有价值，但不必在每个小项目都强行套上这些思维 |

## 用法{#usage}

你可以通过 `@Injectable()` 装饰器的参数选项来指定一个服务的注入作用域，例如将其声明为请求级（`Scope.REQUEST`）：

```ts
import { Injectable, Scope } from '@nestjs/common';

@Injectable({ scope: Scope.REQUEST })
export class CatsService {}
```

- `Scope.REQUEST`: 每个请求实例化一次，适用于需要请求隔离的场景
- `Scope.DEFAULT`: 默认值，表示全局单例（不写也默认是）
- `Scope.TRANSIENT`: 每次注入都会新建一个实例（更少使用）

**自定义提供者（Custom Provider）设置作用域**

除了使用 `@Injectable()` 方式，Nest 也支持**手动注册提供者**（custom provider）。这在你需要更灵活的方式注册某个类时使用，比如指定 `useClass` 或 `useFactory`。

这时作用域设置的写法如下：

```ts
 providers: [
    {
      provide: AppServiceREQUEST, // 提供者名称
      useClass: AppServiceREQUEST, // 提供者类
      scope: Scope.TRANSIENT, // 作用域
    }
 ]
```

这里等价于说：“我注册一个叫 `AppServiceREQUEST` 的服务，使用 `AppServiceREQUEST` 类来创建实例，每次注入都创建一个新实例（瞬态）。”

**特殊注意事项：WebSocket 与请求作用域不兼容**

WebSocket 网关（Gateway）**不能使用请求作用域的服务**，因为它本身是一个 **长期存在的单例对象**。

具体来说：

- WebSocket 是一个**长连接**协议，一个客户端连接可能存在很长时间；
- WebSocket Gateway 是以单例的形式存在的，它保存了每个连接的 socket 实例；
- 请求作用域服务是**每次请求重建的**，这和长连接逻辑冲突；
- 因此，如果你试图在 WebSocket 中注入请求级服务，Nest 会报错或表现异常。

**同样的限制**也适用于其他一些特殊场景中的提供者，比如：

| 特殊组件                 | 原因说明                               |
| ------------------------ | -------------------------------------- |
| WebSocket 网关           | 长连接，要求单例                       |
| Passport 策略（Auth）    | 注册为全局守卫，需要稳定引用           |
| `@Cron()` 定时任务控制器 | 由调度器周期性调用，不走 HTTP 请求流程 |

## 控制器作用域{#controller-scope}

在 NestJS 中，不只是服务（Provider）可以设置作用域，**控制器（Controller）同样可以设置作用域**。作用域决定了控制器的**生命周期管理方式**，即 Nest 如何实例化和销毁它。

**默认行为**

- 默认情况下，控制器是 **单例（`Scope.DEFAULT`）**，在应用启动时创建一次，**所有请求共用一个实例**。
- 这种模式适用于大部分无状态控制器，例如标准 REST API 控制器。

**请求级作用域控制器（`Scope.REQUEST`）**

如果你需要让控制器的实例在每个请求之间隔离（每次请求一个新实例），你可以为控制器设置请求作用域：

```ts
import { Controller, Scope } from '@nestjs/common';

@Controller({
  path: 'cats',
  scope: Scope.REQUEST,
})
export class CatsController {}
```

- 在这种模式下，Nest 会为每个入站请求创建新的 `CatsController` 实例；
- 请求处理完成后，实例会被自动销毁（垃圾回收）；
- 所有控制器内的方法（例如 `@Get()`、`@Post()`）都会绑定到这个隔离的请求上下文。

**注意事项**

1. **控制器作用域影响整个控制器实例**，但不会影响注入的服务，服务的作用域仍由其自身定义的 `@Injectable()` 决定；
2. **不要在请求级控制器中注入单例服务并尝试改变其状态**，否则可能发生跨请求状态污染；
3. 控制器设置为 `Scope.REQUEST` 后，其构造函数中注入的所有请求级服务将自动获得相同请求上下文；
4. 请求级控制器的性能比单例控制器稍差，因为实例需要频繁创建与销毁，建议**按需使用**。

**可能的场景**

| 场景                         | 原因说明                                          |
| ---------------------------- | ------------------------------------------------- |
| 多租户逻辑                   | 需要根据当前请求上下文（如 tenantId）隔离服务实例 |
| 请求上下文追踪（如 traceId） | 在控制器内持有每个请求独立的数据                  |
| 请求级依赖注入               | 控制器依赖某个请求级服务，必须在相同作用域内工作  |

## 作用域层次结构{#scope-hierarchy}

NestJS 的作用域（`Scope`）系统，不仅控制**服务/控制器的生命周期**，还在**依赖注入链中**具有传播特性，尤其是 `REQUEST` 和 `TRANSIENT` 作用域。

REQUEST 作用域：**“向上冒泡”传播**

如果你在一个注入链中，将某个服务设置为 `Scope.REQUEST`，那么它的**所有依赖者也会自动成为请求级作用域**，以保证作用域一致性。

例子：

```ts
CatsController  <--  CatsService  <--  CatsRepository
```

- `CatsService`: 设置为 `@Injectable({ scope: Scope.REQUEST })`
- `CatsController`: 即使你没有设置作用域，也会**自动变成请求级**，因为它依赖了一个请求级服务
- `CatsRepository`: 如果是默认作用域（单例），则仍然保持单例，因为它不依赖请求级服务

```ts
@Injectable({ scope: Scope.REQUEST })
export class CatsService {
  constructor(private readonly repo: CatsRepository) {}
}

@Controller('cats') // 自动变为 REQUEST 作用域
export class CatsController {
  constructor(private readonly catsService: CatsService) {}
}
```

NestJS 会自动追踪注入链，**只要某个环节是请求级，往上全部变成请求级，往下不变**。

**注意**：这种“冒泡”行为只适用于 `Scope.REQUEST`，**不是所有作用域都会向上传播！**

TRANSIENT 作用域：**不会向上传播**

瞬态作用域（`Scope.TRANSIENT`）的行为不同 — 它**不会影响其注入者**的作用域：

示例结构：

```ts
DogsService <-- LoggerService
```

- `LoggerService`: 被设置为瞬态作用域（`Scope.TRANSIENT`）
- `DogsService`: 是默认单例（没有设置作用域）
- Nest 的行为：每次注入 `LoggerService` 都会新建一个实例，但 `DogsService` 自己仍然是单例！

```ts
@Injectable({ scope: Scope.TRANSIENT })
export class LoggerService {}

@Injectable()
export class DogsService {
  constructor(private readonly logger: LoggerService) {}
}
```

`DogsService` 保持单例，而 `LoggerService` 每次注入时都新建。

如果你希望**DogsService 每次注入时也新建一个实例**（即自身也是瞬态），你必须：

```ts
@Injectable({ scope: Scope.TRANSIENT })
export class DogsService {
  constructor(private readonly logger: LoggerService) {}
}
```

Nest 不会“反向传播”瞬态行为，必须**显式声明每一层**。

**建议**

- **Nest 会自动帮你处理作用域链的生命周期一致性**，所以多数时候不需要你手动处理控制器作用域。
- **只有在瞬态服务链中**，你需要手动为每个类设置 `Scope.TRANSIENT`。
- 如果你不确定某个服务实例是如何注入的，可以用日志输出 `console.log(this)` 查看作用域行为。

**注意**：一个控制器（Controller）要么是单例，要么是请求级，不能“某些路由方法单例、某些路由方法请求级”混用。

如果我确实需要某些路由使用请求级服务，某些使用单例怎么办？

推荐做法：**将路由拆分到不同控制器中**

```ts
// CatsController (单例)          <-- 只使用单例服务
// CatsRequestScopedController    <-- 使用请求级服务

// 单例控制器
@Controller('cats')
export class CatsController {
  constructor(private readonly catService: CatService) {}

  @Get()
  getAllCats() {
    return this.catService.findAll(); // 单例逻辑
  }
}

// 请求级控制器
@Controller({ path: 'cats/request', scope: Scope.REQUEST })
export class CatsRequestScopedController {
  constructor(private readonly ctx: UserContextService) {}

  @Get('me')
  getCurrentUserCats() {
    return this.ctx.getUserId(); // 请求独立逻辑
  }
}
```

然后在 `AppModule` 注册两个控制器即可：

```ts
@Module({
  controllers: [CatsController, CatsRequestScopedController],
})
export class AppModule {}
```

为什么不能在一个控制器里混用？

NestJS 的依赖注入模型中：

- 控制器的作用域由其依赖的服务决定
- 如果某个方法依赖请求级服务，而其他方法不依赖，Nest **无法区分方法粒度的作用域**，只能以整个类为单位

所以：**只要一个构造函数参数是请求级服务，整个控制器就变成请求级**。

坑：如果你这样写：

```ts
@Controller('cats')
export class CatsController {
  constructor(
    private readonly singletonService: ConfigService,
    private readonly requestService: UserContextService, // 请求级！
  ) {}

  @Get()
  find() {
    return this.singletonService.get('x'); // 看似只用了单例
  }
}
```

即使 `find()` 方法没有用 `requestService`，**控制器还是请求级**！因为 Nest 无法做到“按方法注入。

## 请求提供器{#request-provider}

在基于 HTTP 的应用中（例如使用 `@nestjs/platform-express` 或 `@nestjs/platform-fastify`），你可能希望在使用 **请求作用域 Scope.REQUEST** 的服务时访问原始请求对象（如 headers、IP、user 等信息）。Nest 提供了一种方式：通过注入 `REQUEST` 对象来实现。

```ts
import { Injectable, Scope, Inject } from '@nestjs/common';
import { REQUEST } from '@nestjs/core';
import { Request } from 'express';

@Injectable({ scope: Scope.REQUEST }) // 请求级作用域
export class CatsService {
  constructor(@Inject(REQUEST) private request: Request) {}
  // ...
}
```

`REQUEST` 是 Nest 提供的特殊 token，代表当前请求上下文中的原始 `Express.Request` 对象。

注入它，可以让你读取当前请求的各种数据（headers、params、cookies、user 等）。

这个方式在你**不需要额外封装服务，只想直接读取原始请求信息**时非常有用。

**不需要自己设置作用域为 REQUEST**，因为只要依赖了 `REQUEST`，Nest 会自动将当前服务标记为请求作用域。

**注意**：

`REQUEST` 本质上就是一个请求作用域的提供器，因此不需要手动指定作用域，因为有冒泡机制。

如果你的服务依赖于 `REQUEST`，Nest 会**自动把这个服务也标记为请求作用域**，这个行为是自动且不可更改的。

但请注意：**你不能通过 `REQUEST` 访问其他请求级服务（如 `UserContextService`）的实例**，它只能提供 Express/Fastify 请求对象。

**GraphQL 特例补充**

由于底层平台的差异，在 GraphQL 应用中，**不能使用 `REQUEST`**，而是使用 `CONTEXT`：

```ts
import { CONTEXT } from '@nestjs/graphql';

@Injectable({ scope: Scope.REQUEST })
export class CatsService {
  constructor(@Inject(CONTEXT) private context) {}
}
```

你需要在 `GraphQLModule` 中配置 `context` 对象，把 request 信息加进去：

```ts
GraphQLModule.forRoot({
  context: ({ req }) => ({ request: req }),
});
```

**总结**:

**请求提供器** 让你在请求作用域的服务中，访问 Express/Fastify 的原始 Request 对象，这是在需要读取用户身份、header、IP 等原始信息时非常常见的做法。比起专门封装一个请求上下文服务，它更轻量直接。

## 查询器提供器{#inquirer-provider}

NestJS 提供了一个特殊的注入令牌 `INQUIRER`，它代表了“当前构建（请求）这个提供器的**父类实例**”。
换句话说，如果一个服务（提供器）被其他服务注入，你可以通过 `INQUIRER` 获取到那个调用方服务的实例（或者至少是它的类实例）。

```ts
import { Inject, Injectable, Scope } from '@nestjs/common';
import { INQUIRER } from '@nestjs/core';

@Injectable({ scope: Scope.TRANSIENT }) // 这里用了瞬态作用域，方便每次注入都是新实例
export class HelloService {
  constructor(@Inject(INQUIRER) private parentClass: object) {}

  sayHello(message: string) {
    console.log(`${this.parentClass?.constructor?.name}: ${message}`);
  }
}
```

`HelloService` 注入了 `INQUIRER`，这个 `parentClass` 就是“谁在使用（构建）这个 HelloService”。

在 `sayHello` 里，它打印出调用者的类名和传入的信息。

```ts
import { Injectable } from '@nestjs/common';
import { HelloService } from './hello.service';

@Injectable()
export class AppService {
  constructor(private helloService: HelloService) {}

  getRoot(): string {
    this.helloService.sayHello('My name is getRoot');

    return 'Hello world!';
  }
}
```

`AppService` 注入并调用了 `HelloService` 的方法。

当调用 `AppService#getRoot()` 时，控制台会输出：

```bash
AppService: My name is getRoot
```

**使用场景**：

- **了解调用链或上下文**：有些情况下，一个服务想知道“是谁调用了我”，可以利用 `INQUIRER` 注入的父实例，获取调用者的类型或信息。
- **日志记录**：自动打印调用者的类名，方便调试和追踪日志来源。
- **指标监控**：可以根据调用者不同，做分类统计，比如哪个模块调用了某个服务。
- **高级框架设计**：实现一些框架特性，动态基于调用方做不同处理。

**注意**：

- `INQUIRER` 一般配合 **瞬态（`TRANSIENT`）作用域**一起用，因为你希望每次注入都能正确区分调用者。
- 如果你的服务是单例，且多个地方共享同一个实例，那么 `INQUIRER` 可能不能准确表示具体调用方。
- 这不是特别常用的功能，属于框架内部或高级用法。

## 性能{#performance}

使用请求作用域的提供者会对应用性能产生影响。虽然 Nest 尝试尽可能缓存元数据，但它仍需在每个请求中创建该类的新实例。因此，这会降低平均响应时间和整体基准测试结果。除非提供者必须是请求作用域，否则强烈建议使用默认的单例作用域。

**提示**
尽管听起来可能让人担心，但设计合理的请求作用域应用，响应延迟通常不会超过大约 5% 的增幅。

**请求级作用域不会影响单例的性能，但它自己的“额外开销”会在整体响应中体现出来，稍微拉低整体性能。**

## 耐用提供器{#durable-providers}

如上一节所述，请求作用域的提供器可能会导致性能下降。只要你有一个请求作用域的服务（比如它被注入进控制器或其他服务），**Nest 就会为每个请求重新创建整个依赖链的实例**，请求结束后再进行垃圾回收。

举例来说，如果你有 30,000 个并发请求，同时请求了某个控制器，而该控制器中注入了一个请求作用域服务，那么 Nest 会临时创建 30,000 份控制器和它们的服务实例。

**多租户问题场景**

假设你在做一个多租户应用，有 10 个客户，每个客户都有自己的数据库连接（或 schema）。你想保证 A 客户永远不会访问 B 客户的数据。那么一种办法是：

为每个请求声明一个「请求作用域的数据源提供器」，它读取请求头中的 `x-tenant-id`，再选择对应的数据库连接。

这么做确实能保证隔离，但问题是：

- 你的大多数服务都依赖于这个 `DataSourceService`；
- 那么它们都会**变成请求作用域**；
- Nest 就要为每个请求**重新创建整棵 DI 树**，很影响性能。

**更优方式**

如果你知道这些请求其实可以按某些公共属性聚类（比如 `tenantId`），那么你没必要为每个请求都重新创建服务实例。你只需要：

- 为每个租户创建一棵 DI 子树，所有来自同一租户的请求共享它；
- 这样大幅减少实例化和 GC 的负担。

这就是**持久 DI 子树**（**耐用提供器**）的场景。

[DI 子树和DI 树](./nest-practice-tip#di子树和di树)

**步骤**

**① 定义 Context 策略**

你需要自定义一个上下文策略 `ContextIdStrategy` 来**告诉 Nest：怎么判断请求属于哪个租户**。

```ts
import {
  HostComponentInfo,
  ContextId,
  ContextIdFactory,
  ContextIdStrategy,
} from '@nestjs/core';
import { Request } from 'express';

const tenants = new Map<string, ContextId>();

export class AggregateByTenantContextIdStrategy implements ContextIdStrategy {
  attach(contextId: ContextId, request: Request) {
    const tenantId = request.headers['x-tenant-id'] as string;

    let tenantSubTreeId: ContextId;
    if (tenants.has(tenantId)) {
      tenantSubTreeId = tenants.get(tenantId)!;
    } else {
      tenantSubTreeId = ContextIdFactory.create();
      tenants.set(tenantId, tenantSubTreeId);
    }

    // 若提供器被标记为 durable，则返回 tenantId 对应的上下文；否则返回默认 contextId
    return (info: HostComponentInfo) =>
      info.isTreeDurable ? tenantSubTreeId : contextId;
  }
}
```

 说明：

- `info.isTreeDurable` 是 Nest 用来判断某个服务是否被标记为 durable（耐用）的；
- 如果是，就用 `tenantSubTreeId`，这样就不会重复创建；
- 如果不是，就走默认逻辑（每次都新建）。

**② 可选：注入 payload（例如 tenantId）**

你也可以让 `REQUEST` 注入的不再是 Express 的 req 对象，而是你自定义的 payload，例如：

```ts
return {
  resolve: (info) =>
    info.isTreeDurable ? tenantSubTreeId : contextId,
  payload: { tenantId },
};
```

这将允许你在服务中：

```ts
constructor(@Inject(REQUEST) private readonly req: { tenantId: string }) {}
```

**③ 全局注册你的策略**

你需要在应用启动时注册这个策略（比如在 `main.ts`）：

```ts
import { ContextIdFactory } from '@nestjs/core';
ContextIdFactory.apply(new AggregateByTenantContextIdStrategy());
```

**④ 标记服务为 durable: true**

你需要手动标记哪些服务可以使用租户 DI 子树：

```ts
@Injectable({ scope: Scope.REQUEST, durable: true })
export class CatsService {}
```

或者在自定义提供器中：

```ts
{
  provide: 'foobar',
  useFactory: () => new Foobar(),
  scope: Scope.REQUEST,
  durable: true,
}
```

| 概念             | 含义说明                                           |
| ---------------- | -------------------------------------------------- |
| Durable Provider | 请求作用域服务 + 可复用实例                        |
| 使用场景         | 多租户、每类请求可归类，避免每次都新建整个服务树   |
| 关键点           | 按 `tenantId` 聚合请求上下文，只为不同租户建立一次 |
| 优势             | 显著减少内存创建/GC，避免性能被请求作用域拖垮      |
| 警告             | 不适合有大量租户的 SaaS 应用，Map 缓存可能爆炸     |

**耐用提供器** 是一种**请求级服务的优化手段**，但是现实情况是：**非常少用**，甚至大多数项目一辈子都不会用到：

| 特性                       | 是否常见                                   |
| -------------------------- | ------------------------------------------ |
| `Scope.DEFAULT`（单例）    | ✅ 90% 项目都靠它撑起                       |
| `Scope.REQUEST`（请求级）  | 🔶 少数涉及 traceId、多租户、GraphQL 等才用 |
| `durable: true` + 聚合策略 | ❌ 极少数“高并发多租户项目”才需要           |

**durable 的设计动机总结**：

请求级服务如果依赖多、被很多地方注入，会拉着整个服务树每个请求都新建，成本高。
durable 的设计目的是：**当我们能“分组请求”时（比如按租户 ID 聚合），就不用每个请求都新建一套服务链了。**

这东西你理解一下原理就够了，**真的到了需要用 durable 的时候，你一定已经做了中大型项目**，并且业务架构足够复杂。
