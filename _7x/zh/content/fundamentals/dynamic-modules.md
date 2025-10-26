### 动态模块

[模块章节](/modules) 涵盖了 Nest 模块的基础知识，并简要介绍了[动态模块](https://docs.nestjs.com/modules#dynamic-modules)。本章节将进一步深入探讨动态模块的主题。完成后，您应该能够很好地理解它们是什么，以及如何以及何时使用它们。

#### 介绍

文档 **概述** 部分中的大多数应用程序代码示例都使用了常规或静态模块。模块定义了如 [providers](/providers) 和 [controllers](/controllers) 这样的组件组，它们作为一个整体应用的模块化部分组合在一起。它们为这些组件提供了一个执行上下文或作用域。例如，定义在模块中的提供者对于模块的其他成员是可见的，而无需导出它们。当一个提供者需要在模块外部可见时，它首先从其宿主模块导出，然后被导入到消费它的模块中。

让我们来看一个熟悉的例子。

首先，我们将定义一个 `UsersModule` 来提供并导出 `UsersService`。`UsersModule` 是 `UsersService` 的 **宿主** 模块。

```typescript
import { Module } from '@nestjs/common';
import { UsersService } from './users.service';

@Module({
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}
```

接下来，我们将定义一个 `AuthModule`，它导入 `UsersModule`，使得 `UsersModule` 导出的提供者可以在 `AuthModule` 内部使用：

```typescript
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { UsersModule } from '../users/users.module';

@Module({
  imports: [UsersModule],
  providers: [AuthService],
  exports: [AuthService],
})
export class AuthModule {}
```

这些结构允许我们在 `AuthModule` 中托管的 `AuthService` 中注入 `UsersService`，例如：

```typescript
import { Injectable } from '@nestjs/common';
import { UsersService } from '../users/users.service';

@Injectable()
export class AuthService {
  constructor(private usersService: UsersService) {}
  /*
    使用 this.usersService 的实现
  */
}
```

我们将这称为 **静态** 模块绑定。Nest 需要连接模块的所有信息已经在宿主模块和消费模块中声明。让我们来剖析一下在这个过程中发生了什么。Nest 通过以下方式使 `UsersService` 在 `AuthModule` 中可用：

1. 实例化 `UsersModule`，包括传递性地导入 `UsersModule` 本身所消费的其他模块，并传递性地解析任何依赖关系（参见 [自定义提供者](https://docs.nestjs.com/fundamentals/custom-providers)）。
2. 实例化 `AuthModule`，并使 `UsersModule` 导出的提供者对 `AuthModule` 中的组件可用（就像它们是在 `AuthModule` 中声明的一样）。
3. 在 `AuthService` 中注入 `UsersService` 的实例。

#### 动态模块使用案例

使用静态模块绑定时，消费模块没有机会 **影响** 宿主模块的提供者如何配置。这为什么重要？考虑这样一种情况，我们有一个通用模块，需要在不同的使用场景下表现不同。这类似于许多系统中的“插件”概念，其中通用设施在使用之前需要一些配置。

Nest 中的一个好例子是 **配置模块**。许多应用程序发现通过使用配置模块来外部化配置细节非常有用。这使得在不同部署中动态更改应用程序设置变得容易：例如，为开发人员使用开发数据库，为预发布/测试环境使用预发布数据库等。通过将配置参数的管理委托给配置模块，应用程序源代码保持独立于配置参数。

挑战在于配置模块本身，由于它是通用的（类似于“插件”），需要由其消费模块进行定制。这就是 _动态模块_ 发挥作用的地方。使用动态模块功能，我们可以使配置模块 **动态化**，以便消费模块可以使用 API 在导入时控制如何定制配置模块。

换句话说，动态模块提供了一个 API，用于将一个模块导入到另一个模块，并在导入时定制该模块的属性和行为，而不是使用我们迄今为止看到的静态绑定。

<app-banner-devtools></app-banner-devtools>

#### 配置模块示例

在本节中，我们将使用[配置章节](https://docs.nestjs.com/techniques/configuration#service)示例代码的基本版本。本章结束时的完整版本可作为一个有效的[示例在此处找到](https://github.com/nestjs/nest/tree/master/sample/25-dynamic-modules)。

我们的需求是让 `ConfigModule` 接受一个 `options` 对象来定制它。这是我们想要支持的功能。基础示例将 `.env` 文件的位置硬编码为项目根文件夹。假设我们想要使其可配置，以便您可以在您选择的任何文件夹中管理您的 `.env` 文件。例如，假设您想将各种 `.env` 文件存储在项目根目录下名为 `config` 的文件夹中（即 `src` 的同级文件夹）。您希望在使用 `ConfigModule` 的不同项目中能够选择不同的文件夹。

动态模块使我们能够将参数传递到被导入的模块中，从而改变其行为。让我们看看这是如何工作的。如果我们从消费模块的角度来看这个目标，然后逆向工作，会很有帮助。首先，让我们快速回顾一下 _静态_ 导入 `ConfigModule` 的示例（即一种无法影响导入模块行为的方法）。请密切关注 `@Module()` 装饰器中的 `imports` 数组：

```typescript
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { ConfigModule } from './config/config.module';

@Module({
  imports: [ConfigModule],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

让我们考虑一下 _动态_ 模块导入，我们传入一个配置对象，可能会是什么样子。比较这两个示例中 `imports` 数组的区别：

```typescript
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { ConfigModule } from './config/config.module';

@Module({
  imports: [ConfigModule.register({ folder: './config' })],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

让我们看看上面的动态示例中发生了什么。有哪些组成部分？

1. `ConfigModule` 是一个普通类，因此我们可以推断它必须有一个名为 `register()` 的 **静态方法**。我们知道它是静态的，因为我们在 `ConfigModule` 类上调用它，而不是在类的 **实例** 上调用。注意：我们很快将创建的这个方法可以有任意名称，但按照惯例，我们应该称其为 `forRoot()` 或 `register()`。
2. `register()` 方法是由我们定义的，因此我们可以接受任何我们喜欢的输入参数。在这种情况下，我们将接受一个具有合适属性的简单 `options` 对象，这是典型的情况。
3. 我们可以推断 `register()` 方法必须返回一个类似 `module` 的东西，因为它的返回值出现在熟悉的 `imports` 列表中，我们迄今为止看到该列表包含模块列表。

事实上，我们的 `register()` 方法将返回一个 `DynamicModule`。动态模块只不过是在运行时创建的模块，具有与静态模块完全相同的属性，外加一个名为 `module` 的附加属性。让我们快速回顾一个示例静态模块声明，密切关注传递给装饰器的模块选项：

```typescript
@Module({
  imports: [DogsModule],
  controllers: [CatsController],
  providers: [CatsService],
  exports: [CatsService]
})
```

动态模块必须返回一个具有完全相同接口的对象，外加一个名为 `module` 的附加属性。`module` 属性用作模块的名称，应与模块的类名相同，如下例所示。

> info **提示** 对于动态模块，模块选项对象的所有属性都是可选的，**除了** `module`。

那么静态的 `register()` 方法呢？我们现在可以看到它的工作是返回一个具有 `DynamicModule` 接口的对象。当我们调用它时，我们实际上是在向 `imports` 列表提供一个模块，类似于在静态情况下通过列出模块类名来这样做。换句话说，动态模块 API 只是返回一个模块，但不是固定 `@Module` 装饰器中的属性，而是以编程方式指定它们。

还有一些细节需要覆盖，以帮助完善整个画面：

1. 我们现在可以说明，`@Module()` 装饰器的 `imports` 属性不仅可以接受模块类名（例如，`imports: [UsersModule]`），还可以接受一个 **返回** 动态模块的函数（例如，`imports: [ConfigModule.register(...)]`）。
2. 动态模块本身可以导入其他模块。我们在本例中不会这样做，但如果动态模块依赖于其他模块的提供者，您将使用可选的 `imports` 属性导入它们。同样，这与使用 `@Module()` 装饰器声明静态模块的元数据完全类似。

有了这种理解，我们现在可以看看我们的动态 `ConfigModule` 声明必须是什么样子。让我们尝试一下。

```typescript
import { DynamicModule, Module } from '@nestjs/common';
import { ConfigService } from './config.service';

@Module({})
export class ConfigModule {
  static register(): DynamicModule {
    return {
      module: ConfigModule,
      providers: [ConfigService],
      exports: [ConfigService],
    };
  }
}
```

现在应该清楚各个部分是如何联系在一起的。调用 `ConfigModule.register(...)` 返回一个 `DynamicModule` 对象，其属性基本上与迄今为止我们通过 `@Module()` 装饰器作为元数据提供的属性相同。

> info **提示** 从 `@nestjs/common` 导入 `DynamicModule`。

然而，我们的动态模块还不是很有趣，因为我们还没有引入任何 **配置** 能力，就像我们之前希望做的那样。接下来让我们解决这个问题。

#### 模块配置

定制 `ConfigModule` 行为的明显解决方案是在静态 `register()` 方法中传递一个 `options` 对象，正如我们上面猜测的那样。让我们再次查看消费模块的 `imports` 属性：

```typescript
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { ConfigModule } from './config/config.module';

@Module({
  imports: [ConfigModule.register({ folder: './config' })],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

这很好地处理了将 `options` 对象传递给我们的动态模块。那么我们如何在 `ConfigModule` 中使用那个 `options` 对象呢？让我们考虑一下。我们知道我们的 `ConfigModule` 基本上是提供和导出可注入服务 `ConfigService` 的宿主，供其他提供者使用。实际上，是我们的 `ConfigService` 需要读取 `options` 对象来定制其行为。让我们暂时假设我们知道如何以某种方式将 `options` 从 `register()` 方法获取到 `ConfigService` 中。基于这个假设，我们可以对服务进行一些更改，以根据 `options` 对象的属性定制其行为。（**注意**：暂时，由于我们 _还没有_ 确定如何传递它，我们将只是硬编码 `options`。我们稍后会修复这个问题）。

```typescript
import { Injectable } from '@nestjs/common';
import * as dotenv from 'dotenv';
import * as fs from 'fs';
import * as path from 'path';
import { EnvConfig } from './interfaces';

@Injectable()
export class ConfigService {
  private readonly envConfig: EnvConfig;

  constructor() {
    const options = { folder: './config' };

    const filePath = `${process.env.NODE_ENV || 'development'}.env`;
    const envFile = path.resolve(__dirname, '../../', options.folder, filePath);
    this.envConfig = dotenv.parse(fs.readFileSync(envFile));
  }

  get(key: string): string {
    return this.envConfig[key];
  }
}
```

现在我们的 `ConfigService` 知道如何在我们指定的 `options` 文件夹中找到 `.env` 文件。

我们剩下的任务是以某种方式将 `options` 对象从 `register()` 步骤注入到我们的 `ConfigService` 中。当然，我们将使用 _依赖注入_ 来实现这一点。这是一个关键点，所以请确保您理解它。我们的 `ConfigModule` 提供 `ConfigService`。`ConfigService` 又依赖于仅在运行时提供的 `options` 对象。因此，在运行时，我们需要首先将 `options` 对象绑定到 Nest IoC 容器，然后让 Nest 将其注入到我们的 `ConfigService` 中。请记住，从 **自定义提供者** 章节中，提供者可以 [包含任何值](https://docs.nestjs.com/fundamentals/custom-providers#non-service-based-providers)，不仅仅是服务，所以我们可以使用依赖注入来处理简单的 `options` 对象。

让我们先解决将 options 对象绑定到 IoC 容器的问题。我们在静态 `register()` 方法中完成这个。请记住，我们正在动态构建一个模块，模块的属性之一是其提供者列表。所以我们需要做的是将我们的 options 对象定义为一个提供者。这将使其可注入到 `ConfigService` 中，我们将在下一步中利用这一点。在下面的代码中，请注意 `providers` 数组：

```typescript
import { DynamicModule, Module } from '@nestjs/common';
import { ConfigService } from './config.service';

@Module({})
export class ConfigModule {
  static register(options: Record<string, any>): DynamicModule {
    return {
      module: ConfigModule,
      providers: [
        {
          provide: 'CONFIG_OPTIONS',
          useValue: options,
        },
        ConfigService,
      ],
      exports: [ConfigService],
    };
  }
}
```

现在我们可以通过将 `'CONFIG_OPTIONS'` 提供者注入到 `ConfigService` 中来完成这个过程。回想一下，当我们使用非类令牌定义提供者时，我们需要使用 `@Inject()` 装饰器，[如这里所述](https://docs.nestjs.com/fundamentals/custom-providers#non-class-based-provider-tokens)。

```typescript
import * as dotenv from 'dotenv';
import * as fs from 'fs';
import * as path from 'path';
import { Injectable, Inject } from '@nestjs/common';
import { EnvConfig } from './interfaces';

@Injectable()
export class ConfigService {
  private readonly envConfig: EnvConfig;

  constructor(@Inject('CONFIG_OPTIONS') private options: Record<string, any>) {
    const filePath = `${process.env.NODE_ENV || 'development'}.env`;
    const envFile = path.resolve(__dirname, '../../', options.folder, filePath);
    this.envConfig = dotenv.parse(fs.readFileSync(envFile));
  }

  get(key: string): string {
    return this.envConfig[key];
  }
}
```

最后一点说明：为简单起见，我们上面使用了基于字符串的注入令牌 (`'CONFIG_OPTIONS'`)，但最佳实践是将其定义为单独文件中的常量（或 `Symbol`），并导入该文件。例如：

```typescript
export const CONFIG_OPTIONS = 'CONFIG_OPTIONS';
```

#### 示例

本章代码的完整示例可以在[这里](https://github.com/nestjs/nest/tree/master/sample/25-dynamic-modules)找到。

#### 社区指南

您可能已经看到在 `@nestjs/` 包中使用了像 `forRoot`、`register` 和 `forFeature` 这样的方法，并且可能想知道这些方法之间的区别。对此没有硬性规定，但 `@nestjs/` 包尝试遵循以下准则：

当创建模块时：

- 使用 `register`，您希望使用特定配置来配置一个动态模块，仅供调用模块使用。例如，使用 Nest 的 `@nestjs/axios`：`HttpModule.register({{ '{' }} baseUrl: 'someUrl' {{ '}' }})`。如果在另一个模块中您使用 `HttpModule.register({{ '{' }} baseUrl: 'somewhere else' {{ '}' }})`，它将具有不同的配置。您可以根据需要为任意多个模块执行此操作。

- 使用 `forRoot`，您希望配置一个动态模块一次，并在多个地方重用该配置（尽管可能由于抽象而未知）。这就是为什么有一个 `GraphQLModule.forRoot()`、一个 `TypeOrmModule.forRoot()` 等。

- 使用 `forFeature`，您希望使用动态模块的 `forRoot` 配置，但需要修改一些特定于调用模块需求的配置（例如，该模块应访问哪个存储库，或记录器应使用的上下文）。

所有这些通常都有它们的异步对应方法，`registerAsync`、`forRootAsync` 和 `forFeatureAsync`，含义相同，但也使用 Nest 的依赖注入进行配置。

#### 可配置模块构建器

由于手动创建高度可配置、暴露异步方法（`registerAsync`、`forRootAsync` 等）的动态模块相当复杂，特别是对于新手，Nest 提供了 `ConfigurableModuleBuilder` 类来简化这个过程，并让您只需几行代码即可构建模块“蓝图”。

例如，让我们使用上面用过的示例（`ConfigModule`）并将其转换为使用 `ConfigurableModuleBuilder`。在开始之前，让我们确保创建一个专用接口，表示我们的 `ConfigModule` 接受什么选项。

```typescript
export interface ConfigModuleOptions {
  folder: string;
}
```

有了这个之后，创建一个新的专用文件（与现有的 `config.module.ts` 文件放在一起）并将其命名为 `config.module-definition.ts`。在这个文件中，让我们利用 `ConfigurableModuleBuilder` 来构建 `ConfigModule` 定义。

```typescript
@@filename(config.module-definition)
import { ConfigurableModuleBuilder } from '@nestjs/common';
import { ConfigModuleOptions } from './interfaces/config-module-options.interface';

export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } =
  new ConfigurableModuleBuilder<ConfigModuleOptions>().build();
@@switch
import { ConfigurableModuleBuilder } from '@nestjs/common';

export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } =
  new ConfigurableModuleBuilder().build();
```

现在让我们打开 `config.module.ts` 文件并修改其实现以利用自动生成的 `ConfigurableModuleClass`：

```typescript
import { Module } from '@nestjs/common';
import { ConfigService } from './config.service';
import { ConfigurableModuleClass } from './config.module-definition';

@Module({
  providers: [ConfigService],
  exports: [ConfigService],
})
export class ConfigModule extends ConfigurableModuleClass {}
```

扩展 `ConfigurableModuleClass` 意味着 `ConfigModule` 现在不仅提供 `register` 方法（如前所述，使用自定义实现时），还提供 `registerAsync` 方法，该方法允许消费者异步配置该模块，例如，通过提供异步工厂：

```typescript
@Module({
  imports: [
    ConfigModule.register({ folder: './config' }),
    // 或者也可以：
    // ConfigModule.registerAsync({
    //   useFactory: () => {
    //     return {
    //       folder: './config',
    //     }
    //   },
    //   inject: [...任何额外的依赖...]
    // }),
  ],
})
export class AppModule {}
```

最后，让我们更新 `ConfigService` 类，以注入生成的模块选项的提供者，而不是我们迄今为止使用的 `'CONFIG_OPTIONS'`。

```typescript
@Injectable()
export class ConfigService {
  constructor(@Inject(MODULE_OPTIONS_TOKEN) private options: ConfigModuleOptions) { ... }
}
```

#### 自定义方法键

默认情况下，`ConfigurableModuleClass` 提供 `register` 及其对应的 `registerAsync` 方法。要使用不同的方法名称，请使用 `ConfigurableModuleBuilder#setClassMethodName` 方法，如下所示：

```typescript
@@filename(config.module-definition)
export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } =
  new ConfigurableModuleBuilder<ConfigModuleOptions>().setClassMethodName('forRoot').build();
@@switch
export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } =
  new ConfigurableModuleBuilder().setClassMethodName('forRoot').build();
```

这种构造将指示 `ConfigurableModuleBuilder` 生成一个暴露 `forRoot` 和 `forRootAsync` 的类，而不是原来的方法。示例：

```typescript
@Module({
  imports: [
    ConfigModule.forRoot({ folder: './config' }), // <-- 注意使用 "forRoot" 而不是 "register"
    // 或者也可以：
    // ConfigModule.forRootAsync({
    //   useFactory: () => {
    //     return {
    //       folder: './config',
    //     }
    //   },
    //   inject: [...任何额外的依赖...]
    // }),
  ],
})
export class AppModule {}
```

#### 自定义选项工厂类

由于 `registerAsync` 方法（或 `forRootAsync` 或任何其他名称，取决于配置）允许消费者传递一个解析为模块配置的提供者定义，库消费者可能可以提供一个类来用于构造配置对象。

```typescript
@Module({
  imports: [
    ConfigModule.registerAsync({
      useClass: ConfigModuleOptionsFactory,
    }),
  ],
})
export class AppModule {}
```

默认情况下，此类必须提供返回模块配置对象的 `create()` 方法。但是，如果您的库遵循不同的命名约定，您可以更改该行为，并指示 `ConfigurableModuleBuilder` 期望不同的方法，例如 `createConfigOptions`，使用 `ConfigurableModuleBuilder#setFactoryMethodName` 方法：

```typescript
@@filename(config.module-definition)
export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } =
  new ConfigurableModuleBuilder<ConfigModuleOptions>().setFactoryMethodName('createConfigOptions').build();
@@switch
export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } =
  new ConfigurableModuleBuilder().setFactoryMethodName('createConfigOptions').build();
```

现在，`ConfigModuleOptionsFactory` 类必须暴露 `createConfigOptions` 方法（而不是 `create`）：

```typescript
@Module({
  imports: [
    ConfigModule.registerAsync({
      useClass: ConfigModuleOptionsFactory, // <-- 这个类必须提供 "createConfigOptions" 方法
    }),
  ],
})
export class AppModule {}
```

#### 额外选项

在某些边缘情况下，您的模块可能需要接受额外的选项，这些选项决定了它应该如何行为（一个很好的例子是 `isGlobal` 标志——或者仅仅是 `global`），同时，这些选项不应包含在 `MODULE_OPTIONS_TOKEN` 提供者中（因为它们与该模块内注册的服务/提供者无关，例如，`ConfigService` 不需要知道其宿主模块是否注册为全局模块）。

在这种情况下，可以使用 `ConfigurableModuleBuilder#setExtras` 方法。请参见以下示例：

```typescript
export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } = new ConfigurableModuleBuilder<ConfigModuleOptions>()
  .setExtras(
    {
      isGlobal: true,
    },
    (definition, extras) => ({
      ...definition,
      global: extras.isGlobal,
    }),
  )
  .build();
```

在上面的示例中，传递给 `setExtras` 方法的第一个参数是一个包含“额外”属性默认值的对象。第二个参数是一个函数，该函数接受自动生成的模块定义（带有 `provider`、`exports` 等）和表示额外属性（由消费者指定或使用默认值）的 `extras` 对象。此函数的返回值是修改后的模块定义。在这个具体示例中，我们获取 `extras.isGlobal` 属性并将其分配给模块定义的 `global` 属性（这反过来决定模块是否是全局的，更多信息请阅读[这里](/modules#dynamic-modules)）。

现在，当消费此模块时，可以传入额外的 `isGlobal` 标志，如下所示：

```typescript
@Module({
  imports: [
    ConfigModule.register({
      isGlobal: true,
      folder: './config',
    }),
  ],
})
export class AppModule {}
```

但是，由于 `isGlobal` 被声明为“额外”属性，它将不会在 `MODULE_OPTIONS_TOKEN` 提供者中可用：

```typescript
@Injectable()
export class ConfigService {
  constructor(@Inject(MODULE_OPTIONS_TOKEN) private options: ConfigModuleOptions) {
    // "options" 对象将没有 "isGlobal" 属性
    // ...
  }
}
```

#### 扩展自动生成的方法

如果需要，可以扩展自动生成的静态方法（`register`、`registerAsync` 等），如下所示：

```typescript
import { Module } from '@nestjs/common';
import { ConfigService } from './config.service';
import { ConfigurableModuleClass, ASYNC_OPTIONS_TYPE, OPTIONS_TYPE } from './config.module-definition';

@Module({
  providers: [ConfigService],
  exports: [ConfigService],
})
export class ConfigModule extends ConfigurableModuleClass {
  static register(options: typeof OPTIONS_TYPE): DynamicModule {
    return {
      // 这里添加您的自定义逻辑
      ...super.register(options),
    };
  }

  static registerAsync(options: typeof ASYNC_OPTIONS_TYPE): DynamicModule {
    return {
      // 这里添加您的自定义逻辑
      ...super.registerAsync(options),
    };
  }
}
```

注意使用 `OPTIONS_TYPE` 和 `ASYNC_OPTIONS_TYPE` 类型，这些类型必须从模块定义文件中导出：

```typescript
export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN, OPTIONS_TYPE, ASYNC_OPTIONS_TYPE } = new ConfigurableModuleBuilder<ConfigModuleOptions>().build();
```