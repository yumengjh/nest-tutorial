---
title: 应用架构设计
createTime: 2025/08/27 21:24:17
permalink: /nest/6ku7kvka/
---

## 基础理念

- **SOLID 原则**
- **模块化开发**（可维护、可扩展）
- **关注点分离**（Separation of Concerns）
- **可测试性 & 可替换性**

## 模块划分思路

### CoreModule（核心模块）

- 放全局只初始化一次的内容
- 全局配置（ConfigService）
- 日志（LoggerService）
- 数据库连接（TypeORM/Prisma Provider）
- 不导出，避免循环依赖

### CommonModule（通用模块）

- 工具类、拦截器、过滤器、管道、守卫
- 响应格式化拦截器
- 异常过滤器
- DTO 验证管道
- 可被任何业务模块导入

### SharedModule（共享模块）

- 通用的业务服务
- 比如邮件发送（MailService）、缓存（CacheService）、第三方 API 封装

###  Domain/Feature Modules（领域/功能模块）

- 用户模块（UserModule）
- 订单模块（OrderModule）
- 商品模块（ProductModule）
- 每个模块包含 controller + service + dto + entity
- 遵循单一职责

###  InfrastructureModule（基础设施模块，可选）

- 和外部世界打交道的服务
- 例如：RedisModule、ElasticsearchModule、MessageQueueModule

### FunctionModule（业务功能聚合，可选）

- 把多个领域模块组合成更大的业务能力
- 例如 PaymentModule 可能调用 User + Order + Notification

## 模块依赖规则

- **CoreModule** 只被 `AppModule` 引用
- **CommonModule** 可被任何模块引用
- **FeatureModule** 之间不直接互相引用 → 通过 **Service 接口** or **事件/消息机制**解耦
- **SharedModule** 避免成为“万能模块”

## 实际案例

以一个电商平台为例：

- CoreModule: 数据库、日志、配置
- CommonModule: 全局管道、异常过滤器
- UserModule: 用户管理
- ProductModule: 商品管理
- OrderModule: 订单管理
- PaymentModule: 聚合订单、用户、商品、第三方支付接口
- NotificationModule: 邮件、短信、推送

## DDD 架构

DDD 不仅仅是代码结构，它是一种 **从业务出发** 的设计思想，目标是：
让代码模型和业务领域模型保持一致，避免“业务在文档里，代码里全是 CRUD”。

### 分层架构（经典 DDD 四层）

DDD 一般分为四层，每一层职责非常明确：

### **用户接口层（Interface / API Layer）**

- 对外暴露系统功能（HTTP API / GraphQL / RPC 等）
- NestJS 对应：**Controller 层**
- 只做输入输出的转换，不写业务逻辑

### **应用层（Application Layer）**

- 负责编排业务流程，调用领域层服务
- 事务管理、权限校验、事件发布
- NestJS 对应：**Service（应用服务）**
- 不直接操作数据库

### **领域层（Domain Layer）**

- **系统的核心，DDD 的灵魂**
- 定义领域模型（Entity、Value Object、Aggregate Root、Domain Service）
- 体现业务规则和领域知识
- 不依赖外部框架（保持纯净）

### **基础设施层（Infrastructure Layer）**

- 为上层提供技术支持
- 数据库 ORM（TypeORM/Prisma）、Redis、消息队列、外部 API
- NestJS 对应：**Repository、外部适配器、第三方 SDK 封装**

## 在 NestJS 中落地 DDD

一个典型的 NestJS + DDD 项目目录：

```markdown
src
├── user
│   ├── application        # 应用层（服务）
│   ├── domain             # 领域层（实体、值对象、聚合、领域服务）
│   ├── infrastructure     # 基础设施（repository 实现、数据库适配）
│   └── interface          # 接口层（controller、DTO）
│
├── order
│   ├── application
│   ├── domain
│   ├── infrastructure
│   └── interface
│
└── shared                 # 公共模块（异常、日志、事件总线等）
```

特点：

- 业务逻辑主要在 **domain 层**
- controller 不写业务逻辑，只调用 **application service**
- 数据库逻辑封装在 **repository**，对 domain 透明

## 举个例子（订单下单流程）

Controller (Interface 层)

```ts
@Post('create')
async create(@Body() dto: CreateOrderDto) {
  return this.orderAppService.createOrder(dto);
}
```

Application Service (应用层)

```ts
@Injectable()
export class OrderAppService {
  constructor(private readonly orderDomainService: OrderDomainService) {}

  async createOrder(dto: CreateOrderDto) {
    // 业务编排
    return this.orderDomainService.create(dto.userId, dto.productId, dto.amount);
  }
}
```

Domain Service (领域层)

```ts
@Injectable()
export class OrderDomainService {
  constructor(private readonly orderRepo: OrderRepository) {}

  create(userId: string, productId: string, amount: number) {
    const order = Order.create(userId, productId, amount);
    order.validate(); // 业务规则校验
    return this.orderRepo.save(order);
  }
}
```

Infrastructure（基础设施层）

```ts
export class Order {
  constructor(
    private readonly id: string,
    private readonly userId: string,
    private readonly productId: string,
    private amount: number,
  ) {}

  static create(userId: string, productId: string, amount: number) {
    return new Order(uuid(), userId, productId, amount);
  }

  validate() {
    if (this.amount <= 0) throw new Error('订单金额必须大于 0');
  }
}
```

DDD 架构的特点是 **业务逻辑内聚在领域层**，上层（Controller、Service）只做编排和协调。
NestJS 本身的模块化和依赖注入机制，和 DDD 思路非常契合。

:::important 重要

**误区**：`Service` 就是写业务逻辑的地方（比如 CRUD 全写在一个 Service 里）。

**进阶理解**：`Service` 并不是 *实现业务逻辑的唯一地方*，而是一个 **业务组织者**。

:::

## DDD 的典型分层

1. **Interface（用户接口层 / 控制器层）**
   - 就是你写的 NestJS **Controller**、GraphQL Resolver、gRPC Handler 之类。
   - 负责接收请求（HTTP、WebSocket…），调用应用层，不写业务。
   - **只负责入站适配**。
2. **Application（应用层）**
   - 定义系统对外提供的用例（Use Case）。
   - 这里不会写具体业务逻辑，而是编排领域服务，把多个业务过程串起来。
   - 举例：订单支付 → 校验库存 (InventoryService) → 扣减库存 → 扣钱 → 创建订单 → 发送通知。
   - 主要职责：
     - 协调领域层
     - 事务控制
     - 权限校验、日志记录等
3. **Domain（领域层）**
   - 核心业务逻辑所在。
   - 包含 **实体 (Entity)**、**值对象 (Value Object)**、**领域服务 (Domain Service)**。
   - 这里不关心数据库、不关心 HTTP，只关注业务规则本身。
   - 举例：
     - **订单实体**（有状态：已支付/未支付）
     - **支付服务**（校验支付金额，检查订单状态）
     - **值对象**（金额 Money，不能为负数）
4. **Infrastructure（基础设施层）**
   - 技术细节的实现：数据库、消息队列、缓存、第三方 API 调用等。
   - 通常提供 **Repository** 实现，把数据存取和业务解耦。
   - 在 NestJS 里一般是 `PrismaService`、`TypeORM Repository`、`RedisService`、`KafkaAdapter` 等。
   - 在 Domain 里只定义接口，在 Infrastructure 里写实现。

```markdown
                         ┌───────────────────┐
                         │   Interface 层    │  ← 控制器/入站适配
                         │ (Controller/API)  │
                         └─────────┬─────────┘
                                   │ 调用
                                   ▼
                         ┌───────────────────┐
                         │  Application 层   │  ← 应用服务 (Use Case)
                         │ (App Service)     │
                         │ - 业务编排        │
                         │ - 事务控制        │
                         └─────────┬─────────┘
                                   │ 调用
                                   ▼
                         ┌───────────────────┐
                         │    Domain 层      │  ← 业务核心
                         │ (Entity, VO,      │
                         │  Domain Service)  │
                         │ - 规则 if/else    │
                         │ - 聚合/状态管理    │
                         └─────────┬─────────┘
                                   │ 调用接口
                                   ▼
                         ┌───────────────────┐
                         │ Infrastructure 层 │  ← 技术细节
                         │ (DB, MQ, Cache,   │
                         │  3rd API, Repo)   │
                         └───────────────────┘
```

## Interface，Types/DTO的位置

### 抽象接口

 一般是 **“领域接口”**（例如 Repository、Domain Service 需要依赖的外部服务接口）。

- **放在领域层 (Domain)**
  - 因为接口描述的是**领域的需求**，而不是某个具体技术。
  - 比如领域层需要一个「用户仓储」，它不关心用的是 MySQL、Mongo 还是 Redis。
  - 这个接口属于领域的“所有权”，实现交给 Infrastructure。

例子：

```ts
// src/domain/user/user.repository.ts
export interface UserRepository {
  findById(id: string): Promise<User>;
  save(user: User): Promise<void>;
}
```

然后在基础设施层实现：

```ts
// src/infrastructure/repositories/user.repository.mysql.ts
@Injectable()
export class MysqlUserRepository implements UserRepository {
  async findById(id: string): Promise<User> { ... }
  async save(user: User): Promise<void> { ... }
}
```

原则：**接口属于领域，实现在 Infrastructure**。这样领域层只依赖抽象，不依赖技术细节。

### DTO / 输入输出类型

DTO 是应用层（Application）的东西，**通常不放在领域层**，因为它是为了 **接口通信/数据传输** 服务的。

- **DTO 属于应用层或接口层**，而不是领域层。
- 领域层只关心领域模型（Entity、ValueObject），而不是网络请求/响应格式。

例子：

```ts
// src/application/user/dto/create-user.dto.ts
export class CreateUserDto {
  username: string;
  password: string;
}
```

控制器用它来接收输入：

```ts
@Post()
create(@Body() dto: CreateUserDto) {
  return this.userAppService.create(dto);
}
```

### 通用类型（Shared Types）

一些**纯工具类的类型**（例如 `Result<T>`、`Paginated<T>`），可以单独放在 `shared/` 或 `common/` 文件夹里。

- 这类跟领域无关，也不是 DTO，而是“跨层共用”的通用结构。

例子：

```ts
// src/shared/result.ts
export type Result<T> = {
  success: boolean;
  data?: T;
  error?: string;
};
```

### 总结

**接口 (Repository, Service Contracts) → 放在领域层**
 （因为它描述的是领域需求，保持“高内聚，低耦合”）

**DTO、入参出参 → 放在应用层/接口层**
 （因为它是通信协议的一部分，不属于领域）

**通用工具类型 → 放在 shared/common 层**
 （避免污染领域和应用）

一个典型 Nest + DDD 项目目录

```markdown
src/
  domain/
    user/
      user.entity.ts
      user.repository.ts   <-- 接口
      user.service.ts
  application/
    user/
      user.service.ts
      dto/
        create-user.dto.ts
        update-user.dto.ts
  infrastructure/
    repositories/
      user.repository.mysql.ts   <-- 实现
  interface/
    user.controller.ts
  shared/
    result.ts
    pagination.ts
```

## 电商后端(NestJS + DDD)

一份**电商后端（NestJS + DDD）`src/` 目录完整示例**（包含四层：Interface / Application / Domain / Infrastructure，以及共享与配置等）

::: file-tree icon="colored" title="电商网站后端 src 结构"

- src
  - common/
    - constants/
      - index.ts
    - decorators/
      - index.ts
    - filters/
      - index.ts
    - interceptors/
      - index.ts
    - middlewares/
      - index.ts
    - pipes/
      - index.ts
    - utils/
      - index.ts
    - index.ts
  - config/
    - app.config.ts
    - database.config.ts
    - jwt.config.ts
    - index.ts
  - modules/
    - auth/
      - controllers/
        - auth.controller.ts
      - dtos/
        - login.dto.ts
      - entities/
        - user.entity.ts
      - services/
        - auth.service.ts
      - strategies/
        - jwt.strategy.ts
      - auth.module.ts
    - products/
      - controllers/
        - products.controller.ts
      - dtos/
        - create-product.dto.ts
      - entities/
        - product.entity.ts
      - services/
        - products.service.ts
      - products.module.ts
    - orders/
      - controllers/
        - orders.controller.ts
      - dtos/
        - create-order.dto.ts
      - entities/
        - order.entity.ts
      - services/
        - orders.service.ts
      - orders.module.ts
    - index.ts
  - shared/
    - decorators/
      - roles.decorator.ts
    - guards/
      - roles.guard.ts
    - interfaces/
      - user.interface.ts
    - pipes/
      - parse-int.pipe.ts
    - index.ts
  - domain/
    - entities/
      - user.entity.ts
      - product.entity.ts
      - order.entity.ts
    - repositories/
      - user.repository.ts
      - product.repository.ts
      - order.repository.ts
    - services/
      - domain-user.service.ts
      - domain-product.service.ts
    - interfaces/
      - repository.interface.ts
    - index.ts
  - application/
    - use-cases/
      - create-user.usecase.ts
      - create-order.usecase.ts
      - create-product.usecase.ts
    - services/
      - application.service.ts
    - dtos/
      - user.dto.ts
      - order.dto.ts
      - product.dto.ts
    - index.ts
  - infrastructure/
    - database/
      - typeorm/
        - user.repository.ts
        - product.repository.ts
        - order.repository.ts
    - services/
      - pdf-generator.service.ts
      - email.service.ts
    - index.ts
  - app.module.ts
  - main.ts
  - index.ts

:::

调用与所有权

数据流与依赖方向：
Interface → Application → Domain → (抽象) → Infrastructure(实现)

所有权：
- 领域模型与仓储 *接口* 属于 Domain（高内聚，防技术入侵）
- DTO/视图/请求响应契约属于 Interface/Application
- 技术实现（数据库/缓存/外部SDK）属于 Infrastructure


```flow
st=>start: 请求进入 API
ctrl=>operation: 控制器层 (Controller)
app=>operation: 应用层 (Application Service)
dom=>operation: 领域层 (Domain)
repo=>operation: 仓储接口 (Repository Interface)
infra=>operation: 基础设施层 (Infrastructure)
db=>inputoutput: 数据库 (DB)

st->ctrl->app->dom
dom->repo
repo->infra->db

```