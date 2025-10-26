### 提供者

提供者是 Nest 中的一个基本概念。许多基本的 Nest 类都可以被视为提供者——服务、仓库、工厂、助手等等。提供者的主要思想是它可以被**注入**为依赖项；这意味着对象可以彼此创建各种关系，并且将“连接”这些对象的功能很大程度上委托给 Nest 运行时系统。

<figure><img class="illustrative-image" src="/assets/Components_1.png" /></figure>

在上一章中，我们构建了一个简单的 `CatsController`。控制器应处理 HTTP 请求并将更复杂的任务委托给**提供者**。提供者是普通的 JavaScript 类，在 NestJS 模块中声明为 `providers`。更多信息，请参阅“模块”章节。

> info **提示** 由于 Nest 使得能够以更面向对象的方式设计和组织依赖项，我们强烈建议遵循 [SOLID 原则](https://en.wikipedia.org/wiki/SOLID)。

#### 服务

让我们从创建一个简单的 `CatsService` 开始。该服务将负责数据的存储和检索，并且设计为由 `CatsController` 使用，因此它是一个很好的定义为提供者的候选者。

```typescript
@@filename(cats.service)
import { Injectable } from '@nestjs/common';
import { Cat } from './interfaces/cat.interface';

@Injectable()
export class CatsService {
  private readonly cats: Cat[] = [];

  create(cat: Cat) {
    this.cats.push(cat);
  }

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

  create(cat) {
    this.cats.push(cat);
  }

  findAll() {
    return this.cats;
  }
}
```

> info **提示** 要使用 CLI 创建服务，只需执行 `$ nest g service cats` 命令。

我们的 `CatsService` 是一个具有一个属性和两个方法的基本类。唯一的新特性是它使用了 `@Injectable()` 装饰器。`@Injectable()` 装饰器附加了元数据，声明 `CatsService` 是一个可以由 Nest [IoC](https://en.wikipedia.org/wiki/Inversion_of_control) 容器管理的类。顺便说一下，这个示例还使用了一个 `Cat` 接口，它可能看起来像这样：

```typescript
@@filename(interfaces/cat.interface)
export interface Cat {
  name: string;
  age: number;
  breed: string;
}
```

现在我们有了一个用于检索猫的服务类，让我们在 `CatsController` 中使用它：

```typescript
@@filename(cats.controller)
import { Controller, Get, Post, Body } from '@nestjs/common';
import { CreateCatDto } from './dto/create-cat.dto';
import { CatsService } from './cats.service';
import { Cat } from './interfaces/cat.interface';

@Controller('cats')
export class CatsController {
  constructor(private catsService: CatsService) {}

  @Post()
  async create(@Body() createCatDto: CreateCatDto) {
    this.catsService.create(createCatDto);
  }

  @Get()
  async findAll(): Promise<Cat[]> {
    return this.catsService.findAll();
  }
}
@@switch
import { Controller, Get, Post, Body, Bind, Dependencies } from '@nestjs/common';
import { CatsService } from './cats.service';

@Controller('cats')
@Dependencies(CatsService)
export class CatsController {
  constructor(catsService) {
    this.catsService = catsService;
  }

  @Post()
  @Bind(Body())
  async create(createCatDto) {
    this.catsService.create(createCatDto);
  }

  @Get()
  async findAll() {
    return this.catsService.findAll();
  }
}
```

`CatsService` 通过类构造函数被**注入**。注意 `private` 语法的使用。这种简写允许我们在同一位置同时声明和初始化 `catsService` 成员。

#### 依赖注入

Nest 是围绕通常称为**依赖注入**的强大设计模式构建的。我们建议在官方 [Angular 文档](https://angular.dev/guide/di)中阅读一篇关于这个概念的优秀文章。

在 Nest 中，得益于 TypeScript 的能力，管理依赖项非常容易，因为它们仅通过类型来解析。在下面的示例中，Nest 将通过创建并返回 `CatsService` 的实例来解析 `catsService`（或者，在单例的正常情况下，如果它已经在其他地方被请求过，则返回现有实例）。此依赖项被解析并传递给控制器的构造函数（或分配给指定的属性）：

```typescript
constructor(private catsService: CatsService) {}
```

#### 作用域

提供者通常具有与应用程序生命周期同步的生存期（“作用域”）。当应用程序引导时，每个依赖项都必须被解析，因此每个提供者都必须被实例化。同样，当应用程序关闭时，每个提供者都将被销毁。但是，也有方法使您的提供者生命周期成为**请求作用域**。您可以在[注入作用域](/fundamentals/injection-scopes)章节中阅读更多关于这些技术的信息。

<app-banner-courses></app-banner-courses>

#### 自定义提供者

Nest 有一个内置的控制反转（“IoC”）容器，用于解析提供者之间的关系。此功能是上述依赖注入功能的基础，但实际上比我们目前描述的功能更强大。有几种定义提供者的方法：您可以使用普通值、类以及异步或同步工厂。更多定义提供者的示例可以在[依赖注入](/fundamentals/dependency-injection)章节中找到。

#### 可选提供者

有时，您可能有一些不一定需要解析的依赖项。例如，您的类可能依赖于一个**配置对象**，但如果没有传递任何配置，则应使用默认值。在这种情况下，依赖项变为可选的，因为缺少配置提供者不会导致错误。

要指示提供者是可选的，请在构造函数的签名中使用 `@Optional()` 装饰器。

```typescript
import { Injectable, Optional, Inject } from '@nestjs/common';

@Injectable()
export class HttpService<T> {
  constructor(@Optional() @Inject('HTTP_OPTIONS') private httpClient: T) {}
}
```

请注意，在上面的示例中，我们使用了一个自定义提供者，这就是我们包含 `HTTP_OPTIONS` 自定义**令牌**的原因。前面的示例展示了基于构造函数的注入，通过构造函数中的类来指示依赖关系。您可以在[自定义提供者](/fundamentals/custom-providers)章节中阅读更多关于自定义提供者及其关联令牌的信息。

#### 基于属性的注入

我们目前使用的技术称为基于构造函数的注入，因为提供者是通过构造函数方法注入的。在一些非常特殊的情况下，**基于属性的注入**可能很有用。例如，如果您的顶级类依赖于一个或多个提供者，通过在于类的构造函数中调用 `super()` 将它们一直传递上去可能会非常繁琐。为了避免这种情况，您可以在属性级别使用 `@Inject()` 装饰器。

```typescript
import { Injectable, Inject } from '@nestjs/common';

@Injectable()
export class HttpService<T> {
  @Inject('HTTP_OPTIONS')
  private readonly httpClient: T;
}
```

> warning **警告** 如果您的类没有扩展另一个类，您应该始终优先使用**基于构造函数**的注入。构造函数明确概述了所需的依赖项，并且比使用 `@Inject` 注解的类属性提供更好的可见性。

#### 提供者注册

现在我们已经定义了一个提供者（`CatsService`），并且有了该服务的消费者（`CatsController`），我们需要向 Nest 注册该服务，以便它可以执行注入。我们通过编辑模块文件（`app.module.ts`）并将服务添加到 `@Module()` 装饰器的 `providers` 数组中来实现这一点。

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

现在 Nest 将能够解析 `CatsController` 类的依赖项。

这是我们现在的目录结构应该看起来的样子：

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
<div class="item">cats.service.ts</div>
</div>
<div class="item">app.module.ts</div>
<div class="item">main.ts</div>
</div>
</div>

#### 手动实例化

到目前为止，我们已经讨论了 Nest 如何自动处理解析依赖项的大部分细节。在某些情况下，您可能需要走出内置的依赖注入系统并手动检索或实例化提供者。我们在下面简要讨论两个这样的主题。

要获取现有实例或动态实例化提供者，您可以使用[模块参考](https://docs.nestjs.com/fundamentals/module-ref)。

要在 `bootstrap()` 函数中获取提供者（例如，对于没有控制器的独立应用程序，或在引导期间利用配置服务），请参阅[独立应用程序](https://docs.nestjs.com/standalone-applications)。