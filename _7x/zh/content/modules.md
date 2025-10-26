### 模块 (Modules)

模块是使用 `@Module()` 装饰器注解的类。`@Module()` 装饰器提供了 **Nest** 用来组织应用程序结构的元数据。

<figure><img class="illustrative-image" src="/assets/Modules_1.png" /></figure>

每个应用程序至少有一个模块，即 **根模块**。根模块是 Nest 用来构建 **应用程序图谱** 的起点——这是 Nest 用于解析模块和提供者之间关系与依赖关系的内部数据结构。虽然理论上非常小的应用程序可能只有根模块，但这并非典型情况。我们想强调的是，**强烈** 推荐使用模块作为组织组件的有效方式。因此，对于大多数应用程序，最终的架构将采用多个模块，每个模块封装一组紧密相关的 **功能**。

`@Module()` 装饰器接受一个对象，该对象的属性描述了模块：

|               |                                                                                                                                                                    |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `providers`   | 由 Nest 注入器实例化的提供者，并且至少可以在此模块内共享                                                                                                               |
| `controllers` | 此模块中定义的、需要被实例化的控制器集合                                                                                                                               |
| `imports`     | 导入的模块列表，这些模块导出了此模块所需的提供者                                                                                                                       |
| `exports`     | 由此模块提供的提供者的子集，并且应该在导入此模块的其他模块中可用。您可以使用提供者本身或其令牌（`provide` 值）                                                               |

模块默认情况下 **封装** 了提供者。这意味着无法注入既不是当前模块的直接组成部分，也不是从导入模块导出的提供者。因此，您可以将模块导出的提供者视为模块的公共接口或 API。

#### 功能模块 (Feature modules)

`CatsController` 和 `CatsService` 属于同一个应用程序域。由于它们紧密相关，将它们移入一个功能模块是合理的。功能模块简单地组织与特定功能相关的代码，保持代码有序并建立清晰的边界。这有助于我们管理复杂性并遵循 [SOLID](https://en.wikipedia.org/wiki/SOLID) 原则进行开发，尤其是在应用程序和/或团队规模增长时。

为了演示这一点，我们将创建 `CatsModule`。

```typescript
@@filename(cats/cats.module)
import { Module } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
export class CatsModule {}
```

> info **提示** 要使用 CLI 创建模块，只需执行 `$ nest g module cats` 命令。

上面，我们在 `cats.module.ts` 文件中定义了 `CatsModule`，并将与此模块相关的所有内容都移到了 `cats` 目录中。我们需要做的最后一件事是将此模块导入到根模块（在 `app.module.ts` 文件中定义的 `AppModule`）中。

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [CatsModule],
})
export class AppModule {}
```

这是我们现在的目录结构：

<div class="file-tree">
  <div class="item">src</div>
  <div class="children">
    <div class="item">cats</div>
    <div class="children">
      <div class="item">dto</div>
      <div class="children">
        <div class="item">create-cat.dto.ts</div>
      </div>
      <div class="item">interfaces</div>
      <div class="children">
        <div class="item">cat.interface.ts</div>
      </div>
      <div class="item">cats.controller.ts</div>
      <div class="item">cats.module.ts</div>
      <div class="item">cats.service.ts</div>
    </div>
    <div class="item">app.module.ts</div>
    <div class="item">main.ts</div>
  </div>
</div>

#### 共享模块 (Shared modules)

在 Nest 中，模块默认是 **单例**，因此您可以轻松地在多个模块之间共享任何提供者的同一实例。

<figure><img class="illustrative-image" src="/assets/Shared_Module_1.png" /></figure>

每个模块自动成为一个 **共享模块**。一旦创建，它可以被任何模块重用。假设我们想在几个其他模块之间共享 `CatsService` 的一个实例。为了做到这一点，我们首先需要通过将 `CatsService` 提供者添加到模块的 `exports` 数组中来 **导出** 它，如下所示：

```typescript
@@filename(cats.module)
import { Module } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
  exports: [CatsService]
})
export class CatsModule {}
```

现在，任何导入 `CatsModule` 的模块都可以访问 `CatsService`，并且将与所有其他导入它的模块共享同一个实例。

如果我们在每个需要 `CatsService` 的模块中直接注册它，它确实可以工作，但这会导致每个模块获得自己独立的 `CatsService` 实例。这可能会导致内存使用增加，因为创建了同一服务的多个实例，并且如果服务维护任何内部状态，也可能导致意外行为，例如状态不一致。

通过将 `CatsService` 封装在一个模块（例如 `CatsModule`）中并导出它，我们确保所有导入 `CatsModule` 的模块都重用 `CatsService` 的同一个实例。这不仅减少了内存消耗，还带来了更可预测的行为，因为所有模块共享同一个实例，使得管理共享状态或资源更加容易。这是在像 NestJS 这样的框架中，模块化和依赖注入的关键好处之一——允许服务在整个应用程序中高效共享。

<app-banner-devtools></app-banner-devtools>

#### 模块再导出 (Module re-exporting)

如上所述，模块可以导出其内部的提供者。此外，它们还可以再导出它们导入的模块。在下面的例子中，`CommonModule` 既被导入到 `CoreModule` 中，又从 `CoreModule` 中导出，使得它对于导入 `CoreModule` 的其他模块可用。

```typescript
@Module({
  imports: [CommonModule],
  exports: [CommonModule],
})
export class CoreModule {}
```

#### 依赖注入 (Dependency injection)

模块类也可以 **注入** 提供者（例如，用于配置目的）：

```typescript
@@filename(cats.module)
import { Module } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
export class CatsModule {
  constructor(private catsService: CatsService) {}
}
@@switch
import { Module, Dependencies } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
@Dependencies(CatsService)
export class CatsModule {
  constructor(catsService) {
    this.catsService = catsService;
  }
}
```

然而，由于 [循环依赖](/fundamentals/circular-dependency) 问题，模块类本身不能作为提供者被注入。

#### 全局模块 (Global modules)

如果您不得不在所有地方导入相同的模块集，可能会变得繁琐。与 Nest 不同，[Angular](https://angular.dev) 的 `providers` 是在全局范围内注册的。一旦定义，它们在任何地方都可用。然而，Nest 将提供者封装在模块范围内。如果不首先导入封装模块，您无法在其他地方使用模块的提供者。

当您希望提供一组应该开箱即用、随处可用的提供者（例如，助手、数据库连接等）时，可以使用 `@Global()` 装饰器将模块设置为 **全局**。

```typescript
import { Module, Global } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Global()
@Module({
  controllers: [CatsController],
  providers: [CatsService],
  exports: [CatsService],
})
export class CatsModule {}
```

`@Global()` 装饰器使模块具有全局作用域。全局模块应该 **只注册一次**，通常由根模块或核心模块注册。在上面的例子中，`CatsService` 提供者将无处不在，希望注入该服务的模块不需要在其 imports 数组中导入 `CatsModule`。

> info **提示** 将所有东西都设为全局并不是一个好的设计决策。全局模块旨在减少必要的样板代码。通常，`imports` 数组是使模块 API 对消费者可用的首选方式。

#### 动态模块 (Dynamic modules)

Nest 模块系统包括一个称为 **动态模块** 的强大功能。此功能使您能够轻松创建可定制的模块，这些模块可以动态注册和配置提供者。动态模块在 [这里](/fundamentals/dynamic-modules) 有详细说明。在本章中，我们将简要概述以完成对模块的介绍。

以下是一个 `DatabaseModule` 的动态模块定义示例：

```typescript
@@filename()
import { Module, DynamicModule } from '@nestjs/common';
import { createDatabaseProviders } from './database.providers';
import { Connection } from './connection.provider';

@Module({
  providers: [Connection],
  exports: [Connection],
})
export class DatabaseModule {
  static forRoot(entities = [], options?): DynamicModule {
    const providers = createDatabaseProviders(options, entities);
    return {
      module: DatabaseModule,
      providers: providers,
      exports: providers,
    };
  }
}
@@switch
import { Module } from '@nestjs/common';
import { createDatabaseProviders } from './database.providers';
import { Connection } from './connection.provider';

@Module({
  providers: [Connection],
  exports: [Connection],
})
export class DatabaseModule {
  static forRoot(entities = [], options) {
    const providers = createDatabaseProviders(options, entities);
    return {
      module: DatabaseModule,
      providers: providers,
      exports: providers,
    };
  }
}
```

> info **提示** `forRoot()` 方法可以同步或异步（即通过 `Promise`）返回动态模块。

该模块默认定义了 `Connection` 提供者（在 `@Module()` 装饰器元数据中），但另外——根据传递给 `forRoot()` 方法的 `entities` 和 `options` 对象——暴露了一系列提供者，例如存储库。请注意，动态模块返回的属性 **扩展**（而不是覆盖）了在 `@Module()` 装饰器中定义的基础模块元数据。这就是如何从模块中导出静态声明的 `Connection` 提供者 **和** 动态生成的存储库提供者。

如果您想在全局范围内注册一个动态模块，请将 `global` 属性设置为 `true`。

```typescript
{
  global: true,
  module: DatabaseModule,
  providers: providers,
  exports: providers,
}
```

> warning **警告** 如上所述，将所有东西都设为全局 **并不是一个好的设计决策**。

`DatabaseModule` 可以按以下方式导入和配置：

```typescript
import { Module } from '@nestjs/common';
import { DatabaseModule } from './database/database.module';
import { User } from './users/entities/user.entity';

@Module({
  imports: [DatabaseModule.forRoot([User])],
})
export class AppModule {}
```

如果您想反过来再导出一个动态模块，可以在 exports 数组中省略 `forRoot()` 方法调用：

```typescript
import { Module } from '@nestjs/common';
import { DatabaseModule } from './database/database.module';
import { User } from './users/entities/user.entity';

@Module({
  imports: [DatabaseModule.forRoot([User])],
  exports: [DatabaseModule],
})
export class AppModule {}
```

[动态模块](/fundamentals/dynamic-modules) 章节更详细地涵盖了此主题，并包含一个 [工作示例](https://github.com/nestjs/nest/tree/master/sample/25-dynamic-modules)。

> info **提示** 在 [本章](/fundamentals/dynamic-modules#configurable-module-builder) 中了解如何使用 `ConfigurableModuleBuilder` 构建高度可定制的动态模块。