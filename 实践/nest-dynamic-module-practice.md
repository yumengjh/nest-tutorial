---
title: 动态模块实践
createTime: 2025/07/27 10:34:28
permalink: /nest/ldvxsju1/
---


动态模块允许你根据不同的配置或运行时条件来灵活地设置模块。这对于需要根据环境、用户需求或第三方服务配置来定制行为的模块非常有用。

创建一个简单的动态模块示例，它会根据配置动态创建一个带有特定前缀的消息服务。

**示例场景**

假设我们需要一个通用的消息服务，但是消息的前缀（例如 "DEBUG:", "INFO:" 等）可以根据应用程序的不同部分或特定需求进行动态配置。

**1. 定义模块选项** (message.module-definition.ts)

首先，我们需要定义这个动态模块能够接受哪些配置选项。NestJS 提供了一个 `ConfigurableModuleBuilder` 来简化动态模块的创建，它会为你生成一些必要的类型和令牌。

```typescript
// src/message/message.module-definition.ts
import { ConfigurableModuleBuilder } from '@nestjs/common';

/**
 * @interface MessageModuleOptions
 * @description 定义 MessageModule 动态配置的选项接口
 */
export interface MessageModuleOptions {
  prefix: string; // 消息前缀，例如 "DEBUG:", "INFO:"
}

// NestJS 提供的构建器，用于简化动态模块的创建。
// 它会自动生成ConfigurableModuleClass, MODULE_OPTIONS_TOKEN, OPTIONS_TYPE, ASYNC_OPTIONS_TYPE
// 这些类型和令牌在动态模块中非常重要，因为它们定义了模块的配置结构和提供方式。
export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN, OPTIONS_TYPE, ASYNC_OPTIONS_TYPE } =
  new ConfigurableModuleBuilder<MessageModuleOptions>().build();
```

 2. **创建一个服务** (message.service.ts)

这个服务将使用动态模块提供的配置。它会有一个构造函数来接收注入的配置选项。

```typescript
// src/message/message.service.ts
import { Injectable, Inject } from '@nestjs/common';
import { MessageModuleOptions, MODULE_OPTIONS_TOKEN } from './message.module-definition';

@Injectable()
export class MessageService {
  private readonly prefix: string;

  // 在创建 MessageService 实例时，注入由动态模块提供的配置对象。
  constructor(@Inject(MODULE_OPTIONS_TOKEN) private options: MessageModuleOptions) {
    this.prefix = options.prefix || '[DEFAULT]'; // 使用注入的前缀，如果没有则使用默认值
    console.log(`MessageService initialized with prefix: ${this.prefix}`);
  }

  log(message: string) {
    console.log(`${this.prefix} ${message}`);
  }
}
```

3. **实现动态模块** (message.module.ts)

这是动态模块的核心部分，它继承自 `ConfigurableModuleClass` 并实现了静态方法 `forRoot` 和 `forRootAsync`。

同步注册

```typescript
// src/message/message.module.ts
import { Module } from '@nestjs/common';
import { MessageService } from './message.service';
import {
  ConfigurableModuleClass,  
  OPTIONS_TYPE, // 约定命名，代表 options 的类型
  ASYNC_OPTIONS_TYPE, // 约定命名，代表异步 options 的类型
} from './message.module-definition';

// 动态消息模块
// 它继承自 ConfigurableModuleClass，该类提供了 forRoot 和 forRootAsync 的基础实现。
@Module({
  // providers 和 exports 在 forRoot/forRootAsync 中动态定义
})
export class MessageModule extends ConfigurableModuleClass {
  // ConfigurableModuleClass 已经提供了 forRoot 和 forRootAsync 的声明，
  // 我们只需要在 Module 装饰器中指定 providers 和 exports。

  // 静态方法，用于同步注册动态模块。
  // 当消费者模块导入 MessageModule.register(...) 时，
  // NestJS 会使用这里提供的配置来实例化 MessageService。
  // 注意：这里我们使用了 ConfigurableModuleClass 提供的 register 方法，
  // 它内部会处理 providers 和 exports。

  static register(options: typeof OPTIONS_TYPE): DynamicModule {
    return {
      module: MessageModule,
      providers: [
        {
          provide: MODULE_OPTIONS_TOKEN,
          useValue: options,
        },
        MessageService,
      ],
      exports: [MessageService],
    };
  }
}
```

异步注册

```typescript
  // 静态方法，用于异步注册动态模块。
  // 当配置需要异步获取（例如从数据库或环境变量中加载）时使用。
  // useFactory 允许你定义一个函数返回配置对象，
  // inject 数组指定 useFactory 函数需要依赖的其他 provider。
  // 注意：这里我们使用了 ConfigurableModuleClass 提供的 registerAsync 方法。
  
  static registerAsync(options: typeof ASYNC_OPTIONS_TYPE): DynamicModule {
    return {
      module: MessageModule,
      imports: options.imports, // 如果异步选项需要导入其他模块
      providers: [
        {
          provide: MODULE_OPTIONS_TOKEN,
          useFactory: options.useFactory,
          inject: options.inject || [],
        },
        MessageService,
      ],
      exports: [MessageService],
    };
  }
```

补充说明

```typescript
// ConfigurableModuleClass 已经为我们处理了 register (即 forRoot) 和 registerAsync (即 forRootAsync)
// 的大部分样板代码。它会确保 MODULE_OPTIONS_TOKEN 被正确提供。
// 所以，在 MessageModule 类本身，我们不需要显式地重写 register 或 registerAsync 方法。
// 只需要在 message.module-definition.ts 中使用 build() 方法，并让 MessageModule extends ConfigurableModuleClass 即可。

// 如果不使用 ConfigurableModuleBuilder，手动实现 dynamic module 看起来像这样：
import { DynamicModule, Module, Provider } from '@nestjs/common';
import { MessageService } from './message.service';

export interface MessageModuleOptions {
  prefix: string;
}

export interface MessageModuleAsyncOptions {
  imports?: any[];
  useFactory?: (...args: any[]) => Promise<MessageModuleOptions> | MessageModuleOptions;
  inject?: any[];
}

@Module({})
export class MessageModule {
  static forRoot(options: MessageModuleOptions): DynamicModule {
    return {
      module: MessageModule,
      providers: [
        {
          provide: 'MESSAGE_OPTIONS', // 一个自定义的令牌来提供选项
          useValue: options,
        },
        MessageService,
      ],
      exports: [MessageService],
    };
  }

  static forRootAsync(options: MessageModuleAsyncOptions): DynamicModule {
    const providers: Provider[] = [
      {
        provide: 'MESSAGE_OPTIONS',
        useFactory: options.useFactory,
        inject: options.inject || [],
      },
      MessageService,
    ];

    return {
      module: MessageModule,
      imports: options.imports,
      providers: providers,
      exports: [MessageService],
    };
  }
}
```

 4. **在消费者模块中使用** (app.module.ts)

现在我们可以在主模块或任何其他模块中导入和使用这个动态模块了。

使用 `register`（同步配置）

在 `app.module.ts` 中导入 `MessageModule.register()`:

```typescript
// src/app.module.ts
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { MessageModule } from './message/message.module'; // 导入我们创建的动态模块

@Module({
  imports: [
    // 这里的 .register() 方法接受一个对象，其结构由 MessageModuleOptions 接口定义。
    MessageModule.register({
      prefix: 'DEBUG', // 这里动态传入了消息前缀
    }),
  ],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

使用 `registerAsync`（异步配置）

如果你需要从环境变量、配置文件或数据库中异步加载配置，可以使用 `registerAsync`。

```typescript
// src/app.module.ts
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { MessageModule } from './message/message.module';
import { ConfigModule, ConfigService } from '@nestjs/config'; // 假设你使用了 @nestjs/config

@Module({
  imports: [
    // 导入 @nestjs/config 模块以获取环境变量
    ConfigModule.forRoot({
      isGlobal: true, // 使 ConfigService 在整个应用中可用
    }),
    // 如果 useFactory 依赖于其他模块的服务，需要在这里导入这些模块。
    // 一个工厂函数，返回一个 Promise 或一个配置对象。
    // 这个函数会在模块初始化时被调用以获取配置。
    // 依赖注入的令牌数组，其对应的服务实例会作为参数传递给 useFactory。
    MessageModule.registerAsync({
      imports: [ConfigModule], // 因为 useFactory 依赖 ConfigService，所以需要导入 ConfigModule
      useFactory: async (configService: ConfigService) => ({
        prefix: configService.get<string>('MESSAGE_PREFIX_ASYNC') || 'ASYNC_DEFAULT:', // 从环境变量中获取前缀
      }),
      inject: [ConfigService], // 注入 ConfigService 到 useFactory
    }),
  ],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```
**注意：** 对于 `registerAsync` 示例，你需要安装 `@nestjs/config` 包 (`npm install @nestjs/config`)，并且在你的 `.env` 文件或环境变量中设置 `MESSAGE_PREFIX_ASYNC`，例如 `MESSAGE_PREFIX_ASYNC=MY_APP_LOG:`。

5. **在控制器中测试** (app.controller.ts)

最后，我们可以在任何地方注入 `MessageService` 并使用它，它将自动使用你在 `AppModule` 中配置的前缀。

```typescript
// src/app.controller.ts
import { Controller, Get } from '@nestjs/common';
import { AppService } from './app.service';
import { MessageService } from './message/message.service'; // 导入 MessageService

@Controller()
export class AppController {
  constructor(
    private readonly appService: AppService,
    private readonly messageService: MessageService, // 注入动态配置的 MessageService
  ) {}

  @Get()
  getHello(): string {
    this.messageService.log('This is a test message from the controller!'); // 使用动态配置的服务
    return this.appService.getHello();
  }
}
```


**总结**

这个例子展示了一个 NestJS 动态模块如何根据配置动态地提供一个服务实例。

*   **`MessageModuleOptions`**: 定义了动态模块接受的配置数据结构。
*   **`ConfigurableModuleBuilder`**: NestJS 提供的工具，大大简化了动态模块的创建过程，为你生成了必要的魔术字符串（`MODULE_OPTIONS_TOKEN`）和基类 (`ConfigurableModuleClass`)。
*   **`MessageService`**: 在构造函数中使用 `@Inject(MODULE_OPTIONS_TOKEN)` 来接收动态注入的配置。这是实现动态行为的关键。
*   **`MessageModule`**: 继承 `ConfigurableModuleClass`，自动获得了 `register` (同步) 和 `registerAsync` (异步) 静态方法，用于在其他模块中导入时指定配置。
*   **使用方式**: 在 `AppModule` 中通过 `MessageModule.register(...)` 或 `MessageModule.registerAsync(...)` 导入模块，并传入你的特定配置然后你就可以根据不同的需求或环境，灵活地配置和重用同一个模块，而不需要为每种配置创建单独的模块。

融合 NestJS 日志输出的服务类：

```typescript
import { MessageModuleOptions, MODULE_OPTIONS_TOKEN } from './message.module-definition';
import { Injectable, Inject } from '@nestjs/common';
import { Logger } from '@nestjs/common';



@Injectable()
export class MessageService {
    private readonly prefix: string;
    private readonly logger: Logger;


    constructor(@Inject(MODULE_OPTIONS_TOKEN) private readonly options: MessageModuleOptions) {
        this.prefix = options.prefix || 'DEFAULT';
        this.logger = new Logger(this.prefix);

    }
    log(message: string) {
        this.logger.verbose(`${message}`);
    }
}
```

模拟异步加载配置项：

```typescript
   static async loggerText(): Promise<string> {
        return new Promise((resolve) => {
            setTimeout(() => {
                resolve('🔨 DEBUG');
            }, 1000);
        });
    }
```

然后在 `AppModule` 中使用此异步配置：

```typescript
 MessageModule.registerAsync({
      imports: [ConfigModule],  
      useFactory: async (configService: ConfigService) => ({
        prefix: await DatabaseService.loggerText(), 
      }),
      inject: [ConfigService],  
    })
```