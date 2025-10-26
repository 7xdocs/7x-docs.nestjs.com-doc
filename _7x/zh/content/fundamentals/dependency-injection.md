### 自定义提供者

在之前的章节中，我们触及了 **依赖注入（DI）** 的各个方面以及它在 Nest 中的使用方式。其中一个例子是基于构造函数的依赖注入，用于将实例（通常是服务提供者）注入到类中。您不会惊讶地发现依赖注入是以一种基本方式内置于 Nest 核心的。到目前为止，我们只探索了一种主要模式。随着应用程序变得越来越复杂，您可能需要利用 DI 系统的全部功能，所以让我们更详细地探索它们。

#### DI 基础

依赖注入是一种控制反转（IoC）技术，您将依赖项的实例化委托给 IoC 容器（在我们的例子中是 NestJS 运行时系统），而不是在您自己的代码中强制进行。让我们来研究一下提供者章节中的这个例子发生了什么。

首先，我们定义一个提供者。`@Injectable()` 装饰器将 `CatsService` 类标记为一个提供者。

```typescript
@@filename(cats.service)
import { Injectable } from '@nestjs/common';
import { Cat } from './interfaces/cat.interface';

@Injectable()
export class CatsService {
  private readonly cats: Cat[] = [];

  findAll(): Cat[] {
    return this.cats;
  }
}
@@switch
import { Injectable } from '@nestjs/common';

@Injectable()
export class CatsService {
  constructor() {
    this.cats = [];
  }

  findAll() {
    return this.cats;
  }
}
```

然后我们请求 Nest 将提供者注入到我们的控制器类中：

```typescript
@@filename(cats.controller)
import { Controller, Get } from '@nestjs/common';
import { CatsService } from './cats.service';
import { Cat } from './interfaces/cat.interface';

@Controller('cats')
export class CatsController {
  constructor(private catsService: CatsService) {}

  @Get()
  async findAll(): Promise<Cat[]> {
    return this.catsService.findAll();
  }
}
@@switch
import { Controller, Get, Bind, Dependencies } from '@nestjs/common';
import { CatsService } from './cats.service';

@Controller('cats')
@Dependencies(CatsService)
export class CatsController {
  constructor(catsService) {
    this.catsService = catsService;
  }

  @Get()
  async findAll() {
    return this.catsService.findAll();
  }
}
```

最后，我们将提供者注册到 Nest IoC 容器中：

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { CatsController } from './cats/cats.controller';
import { CatsService } from './cats/cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
export class AppModule {}
```

为了使这一切工作，底层究竟发生了什么？这个过程有三个关键步骤：

1. 在 `cats.service.ts` 中，`@Injectable()` 装饰器声明 `CatsService` 类是一个可以由 Nest IoC 容器管理的类。
2. 在 `cats.controller.ts` 中，`CatsController` 通过构造函数注入声明了对 `CatsService` 令牌的依赖：

```typescript
  constructor(private catsService: CatsService)
```

3. 在 `app.module.ts` 中，我们将令牌 `CatsService` 与来自 `cats.service.ts` 文件的 `CatsService` 类关联起来。我们将在下面看到这种关联（也称为注册）具体是如何发生的。

当 Nest IoC 容器实例化一个 `CatsController` 时，它首先查找所有依赖项\*。当它找到 `CatsService` 依赖项时，它会对 `CatsService` 令牌执行查找，根据注册步骤（上面的第 3 步）返回 `CatsService` 类。假设是 `SINGLETON` 作用域（默认行为），Nest 将然后创建一个 `CatsService` 实例，缓存它并返回它，或者如果已经有一个被缓存了，就返回现有的实例。

\*这个解释稍微简化以说明要点。我们忽略的一个重要领域是，分析代码依赖关系的过程非常复杂，并且发生在应用程序引导期间。一个关键特性是依赖分析（或“创建依赖图”）是**可传递的**。在上面的例子中，如果 `CatsService` 本身有依赖项，那些也会被解析。依赖图确保依赖项以正确的顺序解析 - 基本上是“自底向上”。这种机制使开发人员不必管理如此复杂的依赖图。

<app-banner-courses></app-banner-courses>

#### 标准提供者

让我们仔细看看 `@Module()` 装饰器。在 `app.module` 中，我们声明：

```typescript
@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
```

`providers` 属性接受一个提供者数组。到目前为止，我们通过类名列表提供了这些提供者。实际上，语法 `providers: [CatsService]` 是更完整语法的简写：

```typescript
providers: [
  {
    provide: CatsService,
    useClass: CatsService,
  },
];
```

现在我们看到了这个显式结构，我们可以理解注册过程了。在这里，我们明确地将令牌 `CatsService` 与类 `CatsService` 关联起来。简写符号仅仅是为了简化最常见的使用情况，即令牌用于请求同名类的实例。

#### 自定义提供者

当您的需求超出了*标准提供者*所提供的范围时会发生什么？以下是一些例子：

- 您想创建一个自定义实例，而不是让 Nest 实例化（或返回缓存实例）一个类
- 您想在第二个依赖项中重用现有的类
- 您想用一个模拟版本覆盖一个类以进行测试

Nest 允许您定义自定义提供者来处理这些情况。它提供了几种定义自定义提供者的方法。让我们来逐一了解。

> info **提示** 如果您在依赖解析方面遇到问题，可以设置 `NEST_DEBUG` 环境变量，并在启动期间获取额外的依赖解析日志。

#### 值提供者：`useValue`

`useValue` 语法对于注入常量值、将外部库放入 Nest 容器或用模拟对象替换真实实现非常有用。假设您希望强制 Nest 在测试目的下使用模拟的 `CatsService`。

```typescript
import { CatsService } from './cats.service';

const mockCatsService = {
  /* mock implementation
  ...
  */
};

@Module({
  imports: [CatsModule],
  providers: [
    {
      provide: CatsService,
      useValue: mockCatsService,
    },
  ],
})
export class AppModule {}
```

在这个例子中，`CatsService` 令牌将解析为 `mockCatsService` 模拟对象。`useValue` 需要一个值——在这个例子中，是一个与它要替换的 `CatsService` 类具有相同接口的字面量对象。由于 TypeScript 的结构类型，您可以使用任何具有兼容接口的对象，包括字面量对象或用 `new` 实例化的类实例。

#### 非基于类的提供者令牌

到目前为止，我们使用类名作为我们的提供者令牌（在 `providers` 数组中列出的提供者的 `provide` 属性的值）。这与基于构造函数的注入使用的标准模式相匹配，其中令牌也是类名。（如果这个概念不完全清楚，请回看 DI 基础 以复习令牌）。有时，我们可能希望灵活地使用字符串或符号作为 DI 令牌。例如：

```typescript
import { connection } from './connection';

@Module({
  providers: [
    {
      provide: 'CONNECTION',
      useValue: connection,
    },
  ],
})
export class AppModule {}
```

在这个例子中，我们将一个字符串值的令牌（`'CONNECTION'`）与从外部文件导入的预先存在的 `connection` 对象关联起来。

> warning **注意** 除了使用字符串作为令牌值，您还可以使用 JavaScript 符号或 TypeScript 枚举。

我们之前已经看到了如何使用标准的基于构造函数的注入模式来注入一个提供者。这种模式要求依赖项用类名声明。`'CONNECTION'` 自定义提供者使用一个字符串值的令牌。让我们看看如何注入这样的提供者。为此，我们使用 `@Inject()` 装饰器。这个装饰器接受一个参数——令牌。

```typescript
@@filename()
@Injectable()
export class CatsRepository {
  constructor(@Inject('CONNECTION') connection: Connection) {}
}
@@switch
@Injectable()
@Dependencies('CONNECTION')
export class CatsRepository {
  constructor(connection) {}
}
```

> info **提示** `@Inject()` 装饰器是从 `@nestjs/common` 包导入的。

虽然我们在上面的例子中直接使用字符串 `'CONNECTION'` 用于说明目的，但为了清晰的代码组织，最佳实践是在单独的文件中定义令牌，例如 `constants.ts`。像对待在自己文件中定义并在需要时导入的符号或枚举一样对待它们。

#### 类提供者：`useClass`

`useClass` 语法允许您动态确定一个令牌应该解析到的类。例如，假设我们有一个抽象（或默认）的 `ConfigService` 类。根据当前环境，我们希望 Nest 提供配置服务的不同实现。以下代码实现了这样的策略。

```typescript
const configServiceProvider = {
  provide: ConfigService,
  useClass:
    process.env.NODE_ENV === 'development'
      ? DevelopmentConfigService
      : ProductionConfigService,
};

@Module({
  providers: [configServiceProvider],
})
export class AppModule {}
```

让我们看一下这个代码示例中的几个细节。您会注意到我们首先用一个字面量对象定义 `configServiceProvider`，然后在模块装饰器的 `providers` 属性中传递它。这只是一点代码组织，但在功能上等同于我们在本章中迄今使用的例子。

另外，我们使用了 `ConfigService` 类名作为我们的令牌。对于任何依赖于 `ConfigService` 的类，Nest 将注入所提供类（`DevelopmentConfigService` 或 `ProductionConfigService`）的实例，覆盖可能在其他地方声明的任何默认实现（例如，用 `@Injectable()` 装饰器声明的 `ConfigService`）。

#### 工厂提供者：`useFactory`

`useFactory` 语法允许动态创建提供者。实际的提供者将由工厂函数返回的值提供。工厂函数可以根据需要简单或复杂。一个简单的工厂可能不依赖于任何其他提供者。一个更复杂的工厂本身可以注入它需要计算结果的其它提供者。对于后一种情况，工厂提供者语法有一对相关的机制：

1. 工厂函数可以接受（可选的）参数。
2. （可选的）`inject` 属性接受一个提供者数组，Nest 将在实例化过程中解析这些提供者并将它们作为参数传递给工厂函数。此外，这些提供者可以被标记为可选的。这两个列表应该相互关联：Nest 将按照相同的顺序将 `inject` 列表中的实例作为参数传递给工厂函数。下面的例子演示了这一点。

```typescript
@@filename()
const connectionProvider = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider: MyOptionsProvider, optionalProvider?: string) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options);
  },
  inject: [MyOptionsProvider, { token: 'SomeOptionalProvider', optional: true }],
  //       \______________/             \__________________/
  //        此提供者是必须的。             此令牌的提供者
  //                                    可以解析为 `undefined`。
};

@Module({
  providers: [
    connectionProvider,
    MyOptionsProvider, // 基于类的提供者
    // { provide: 'SomeOptionalProvider', useValue: 'anything' },
  ],
})
export class AppModule {}
@@switch
const connectionProvider = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider, optionalProvider) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options);
  },
  inject: [MyOptionsProvider, { token: 'SomeOptionalProvider', optional: true }],
  //       \______________/            \__________________/
  //        此提供者是必须的。            此令牌的提供者
  //                                   可以解析为 `undefined`。
};

@Module({
  providers: [
    connectionProvider,
    MyOptionsProvider, // 基于类的提供者
    // { provide: 'SomeOptionalProvider', useValue: 'anything' },
  ],
})
export class AppModule {}
```

#### 别名提供者：`useExisting`

`useExisting` 语法允许您为现有的提供者创建别名。这创造了两种访问同一提供者的方式。在下面的例子中，（基于字符串的）令牌 `'AliasedLoggerService'` 是（基于类的）令牌 `LoggerService` 的别名。假设我们有两个不同的依赖项，一个用于 `'AliasedLoggerService'`，一个用于 `LoggerService`。如果两个依赖项都指定为 `SINGLETON` 作用域，它们将解析到同一个实例。

```typescript
@Injectable()
class LoggerService {
  /* implementation details */
}

const loggerAliasProvider = {
  provide: 'AliasedLoggerService',
  useExisting: LoggerService,
};

@Module({
  providers: [LoggerService, loggerAliasProvider],
})
export class AppModule {}
```

#### 非基于服务的提供者

虽然提供者通常提供服务，但它们并不限于这种用法。一个提供者可以提供任何值。例如，一个提供者可以根据当前环境提供一个配置对象数组，如下所示：

```typescript
const configFactory = {
  provide: 'CONFIG',
  useFactory: () => {
    return process.env.NODE_ENV === 'development' ? devConfig : prodConfig;
  },
};

@Module({
  providers: [configFactory],
})
export class AppModule {}
```

#### 导出自定义提供者

像任何提供者一样，自定义提供者的作用域限于其声明模块。要使它对其他模块可见，必须将其导出。要导出自定义提供者，我们可以使用其令牌或完整的提供者对象。

以下示例展示了使用令牌导出：

```typescript
@@filename()
const connectionFactory = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider: OptionsProvider) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options);
  },
  inject: [OptionsProvider],
};

@Module({
  providers: [connectionFactory],
  exports: ['CONNECTION'],
})
export class AppModule {}
@@switch
const connectionFactory = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options);
  },
  inject: [OptionsProvider],
};

@Module({
  providers: [connectionFactory],
  exports: ['CONNECTION'],
})
export class AppModule {}
```

或者，使用完整的提供者对象导出：

```typescript
@@filename()
const connectionFactory = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider: OptionsProvider) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options);
  },
  inject: [OptionsProvider],
};

@Module({
  providers: [connectionFactory],
  exports: [connectionFactory],
})
export class AppModule {}
@@switch
const connectionFactory = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options);
  },
  inject: [OptionsProvider],
};

@Module({
  providers: [connectionFactory],
  exports: [connectionFactory],
})
export class AppModule {}
```