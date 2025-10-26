### 测试

自动化测试被认为是任何严肃软件开发工作中必不可少的一部分。自动化使得在开发过程中能够轻松快捷地重复运行单个测试或测试套件。这有助于确保发布版本满足质量和性能目标。自动化有助于提高测试覆盖率，并为开发人员提供更快的反馈循环。自动化既提高了单个开发人员的生产力，又确保了在关键的开发生命周期节点（如源代码控制检入、功能集成和版本发布）运行测试。

这类测试通常涵盖多种类型，包括单元测试、端到端（e2e）测试、集成测试等。尽管好处毋庸置疑，但设置它们可能很繁琐。Nest 致力于推广开发最佳实践，包括有效的测试，因此它包含以下特性来帮助开发人员和团队构建和自动化测试。Nest：

- 自动为组件和应用程序的 e2e 测试搭建默认的单元测试
- 提供默认工具（例如构建独立模块/应用程序加载器的测试运行器）
- 开箱即用地提供与 [Jest](https://github.com/facebook/jest) 和 [Supertest](https://github.com/visionmedia/supertest) 的集成，同时保持对测试工具的不可知性
- 在测试环境中提供 Nest 依赖注入系统，以便轻松模拟组件

如前所述，您可以使用任何您喜欢的**测试框架**，因为 Nest 不强加任何特定工具。只需替换所需的元素（例如测试运行器），您仍然可以享受 Nest 现成的测试工具带来的好处。

#### 安装

首先，安装所需的包：

```bash
$ npm i --save-dev @nestjs/testing
```

#### 单元测试

在以下示例中，我们测试两个类：`CatsController` 和 `CatsService`。如前所述，[Jest](https://github.com/facebook/jest) 被提供为默认测试框架。它充当测试运行器，同时还提供断言函数和测试替身实用工具，以帮助进行模拟、监视等。在以下基本测试中，我们手动实例化这些类，并确保控制器和服务履行其 API 约定。

```typescript
@@filename(cats.controller.spec)
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

describe('CatsController', () => {
  let catsController: CatsController;
  let catsService: CatsService;

  beforeEach(() => {
    catsService = new CatsService();
    catsController = new CatsController(catsService);
  });

  describe('findAll', () => {
    it('should return an array of cats', async () => {
      const result = ['test'];
      jest.spyOn(catsService, 'findAll').mockImplementation(() => result);

      expect(await catsController.findAll()).toBe(result);
    });
  });
});
@@switch
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

describe('CatsController', () => {
  let catsController;
  let catsService;

  beforeEach(() => {
    catsService = new CatsService();
    catsController = new CatsController(catsService);
  });

  describe('findAll', () => {
    it('should return an array of cats', async () => {
      const result = ['test'];
      jest.spyOn(catsService, 'findAll').mockImplementation(() => result);

      expect(await catsController.findAll()).toBe(result);
    });
  });
});
```

> info **提示** 将测试文件放在它们测试的类附近。测试文件应具有 `.spec` 或 `.test` 后缀。

由于上述示例很简单，我们并没有真正测试任何特定于 Nest 的内容。实际上，我们甚至没有使用依赖注入（注意我们将 `CatsService` 的实例传递给我们的 `catsController`）。这种测试形式——我们手动实例化被测试的类——通常被称为**独立测试**，因为它独立于框架。让我们介绍一些更高级的功能，帮助您测试更广泛使用 Nest 特性的应用程序。

#### 测试工具

`@nestjs/testing` 包提供了一组实用工具，能够实现更健壮的测试过程。让我们使用内置的 `Test` 类重写前面的示例：

```typescript
@@filename(cats.controller.spec)
import { Test } from '@nestjs/testing';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

describe('CatsController', () => {
  let catsController: CatsController;
  let catsService: CatsService;

  beforeEach(async () => {
    const moduleRef = await Test.createTestingModule({
        controllers: [CatsController],
        providers: [CatsService],
      }).compile();

    catsService = moduleRef.get(CatsService);
    catsController = moduleRef.get(CatsController);
  });

  describe('findAll', () => {
    it('should return an array of cats', async () => {
      const result = ['test'];
      jest.spyOn(catsService, 'findAll').mockImplementation(() => result);

      expect(await catsController.findAll()).toBe(result);
    });
  });
});
@@switch
import { Test } from '@nestjs/testing';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

describe('CatsController', () => {
  let catsController;
  let catsService;

  beforeEach(async () => {
    const moduleRef = await Test.createTestingModule({
        controllers: [CatsController],
        providers: [CatsService],
      }).compile();

    catsService = moduleRef.get(CatsService);
    catsController = moduleRef.get(CatsController);
  });

  describe('findAll', () => {
    it('should return an array of cats', async () => {
      const result = ['test'];
      jest.spyOn(catsService, 'findAll').mockImplementation(() => result);

      expect(await catsController.findAll()).toBe(result);
    });
  });
});
```

`Test` 类有助于提供一个应用程序执行上下文，该上下文本质上模拟了完整的 Nest 运行时，但为您提供了钩子，使得管理类实例（包括模拟和覆盖）变得容易。`Test` 类有一个 `createTestingModule()` 方法，该方法接受一个模块元数据对象作为其参数（与您传递给 `@Module()` 装饰器的对象相同）。此方法返回一个 `TestingModule` 实例，该实例又提供了一些方法。对于单元测试，重要的是 `compile()` 方法。此方法使用其依赖项引导一个模块（类似于在传统的 `main.ts` 文件中使用 `NestFactory.create()` 引导应用程序的方式），并返回一个准备好进行测试的模块。

> info **提示** `compile()` 方法是**异步的**，因此必须等待。一旦模块编译完成，您就可以使用 `get()` 方法检索它声明的任何**静态**实例（控制器和提供者）。

`TestingModule` 继承自[模块引用](/fundamentals/module-ref)类，因此具有动态解析作用域提供者（瞬态或请求作用域）的能力。使用 `resolve()` 方法来实现这一点（`get()` 方法只能检索静态实例）。

```typescript
const moduleRef = await Test.createTestingModule({
  controllers: [CatsController],
  providers: [CatsService],
}).compile();

catsService = await moduleRef.resolve(CatsService);
```

> warning **警告** `resolve()` 方法从它自己的 **DI 容器子树** 返回提供者的唯一实例。每个子树都有一个唯一的上下文标识符。因此，如果您多次调用此方法并比较实例引用，您会发现它们不相等。

> info **提示** 了解更多关于模块引用功能的信息，请参阅[这里](/fundamentals/module-ref)。

您可以使用[自定义提供者](/fundamentals/custom-providers)覆盖任何提供者的生产版本，以用于测试目的。例如，您可以模拟数据库服务，而不是连接到实时数据库。我们将在下一节中介绍覆盖，但它们也可用于单元测试。

<app-banner-courses></app-banner-courses>

#### 自动模拟

Nest 还允许您定义一个模拟工厂，应用于所有缺失的依赖项。这在类中有大量依赖项且模拟所有这些依赖项将花费很长时间和大量设置的情况下非常有用。要使用此功能，`createTestingModule()` 需要与 `useMocker()` 方法链式调用，传递一个用于依赖项模拟的工厂。该工厂可以接受一个可选的令牌，这是一个实例令牌，任何对 Nest 提供者有效的令牌，并返回一个模拟实现。以下是一个使用 [`jest-mock`](https://www.npmjs.com/package/jest-mock) 创建通用模拟器和使用 `jest.fn()` 为 `CatsService` 创建特定模拟的示例。

```typescript
// ...
import { ModuleMocker, MockFunctionMetadata } from 'jest-mock';

const moduleMocker = new ModuleMocker(global);

describe('CatsController', () => {
  let controller: CatsController;

  beforeEach(async () => {
    const moduleRef = await Test.createTestingModule({
      controllers: [CatsController],
    })
      .useMocker((token) => {
        const results = ['test1', 'test2'];
        if (token === CatsService) {
          return { findAll: jest.fn().mockResolvedValue(results) };
        }
        if (typeof token === 'function') {
          const mockMetadata = moduleMocker.getMetadata(
            token,
          ) as MockFunctionMetadata<any, any>;
          const Mock = moduleMocker.generateFromMetadata(mockMetadata);
          return new Mock();
        }
      })
      .compile();

    controller = moduleRef.get(CatsController);
  });
});
```

您也可以像通常获取自定义提供者一样从测试容器中获取这些模拟，`moduleRef.get(CatsService)`。

> info **提示** 通用的模拟工厂，如 [`@golevelup/ts-jest`](https://github.com/golevelup/nestjs/tree/master/packages/testing) 中的 `createMock`，也可以直接传递。

> info **提示** `REQUEST` 和 `INQUIRER` 提供者不能被自动模拟，因为它们已经在上下文中预定义。但是，可以使用自定义提供者语法或利用 `.overrideProvider` 方法**覆盖**它们。

#### 端到端测试

与专注于单个模块和类的单元测试不同，端到端（e2e）测试在更聚合的级别上覆盖类和模块的交互——更接近最终用户与生产系统的交互类型。随着应用程序的增长，手动测试每个 API 端点的端到端行为变得困难。自动化的端到端测试帮助我们确保系统的整体行为是正确的并满足项目需求。为了执行 e2e 测试，我们使用与**单元测试**中类似的配置。此外，Nest 使得使用 [Supertest](https://github.com/visionmedia/supertest) 库来模拟 HTTP 请求变得容易。

```typescript
@@filename(cats.e2e-spec)
import * as request from 'supertest';
import { Test } from '@nestjs/testing';
import { CatsModule } from '../../src/cats/cats.module';
import { CatsService } from '../../src/cats/cats.service';
import { INestApplication } from '@nestjs/common';

describe('Cats', () => {
  let app: INestApplication;
  let catsService = { findAll: () => ['test'] };

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({
      imports: [CatsModule],
    })
      .overrideProvider(CatsService)
      .useValue(catsService)
      .compile();

    app = moduleRef.createNestApplication();
    await app.init();
  });

  it(`/GET cats`, () => {
    return request(app.getHttpServer())
      .get('/cats')
      .expect(200)
      .expect({
        data: catsService.findAll(),
      });
  });

  afterAll(async () => {
    await app.close();
  });
});
@@switch
import * as request from 'supertest';
import { Test } from '@nestjs/testing';
import { CatsModule } from '../../src/cats/cats.module';
import { CatsService } from '../../src/cats/cats.service';
import { INestApplication } from '@nestjs/common';

describe('Cats', () => {
  let app: INestApplication;
  let catsService = { findAll: () => ['test'] };

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({
      imports: [CatsModule],
    })
      .overrideProvider(CatsService)
      .useValue(catsService)
      .compile();

    app = moduleRef.createNestApplication();
    await app.init();
  });

  it(`/GET cats`, () => {
    return request(app.getHttpServer())
      .get('/cats')
      .expect(200)
      .expect({
        data: catsService.findAll(),
      });
  });

  afterAll(async () => {
    await app.close();
  });
});
```

> info **提示** 如果您使用 [Fastify](/techniques/performance) 作为 HTTP 适配器，它需要稍微不同的配置，并且具有内置的测试能力：
>
> ```ts
> let app: NestFastifyApplication;
>
> beforeAll(async () => {
>   app = moduleRef.createNestApplication<NestFastifyApplication>(
>     new FastifyAdapter(),
>   );
>
>   await app.init();
>   await app.getHttpAdapter().getInstance().ready();
> });
>
> it(`/GET cats`, () => {
>   return app
>     .inject({
>       method: 'GET',
>       url: '/cats',
>     })
>     .then((result) => {
>       expect(result.statusCode).toEqual(200);
>       expect(result.payload).toEqual(/* expectedPayload */);
>     });
> });
>
> afterAll(async () => {
>   await app.close();
> });
> ```

在这个例子中，我们基于前面描述的一些概念进行构建。除了我们之前使用的 `compile()` 方法，我们现在使用 `createNestApplication()` 方法来实例化一个完整的 Nest 运行时环境。

需要考虑的一个注意事项是，当您的应用程序使用 `compile()` 方法编译时，`HttpAdapterHost#httpAdapter` 在那时将是未定义的。这是因为在此编译阶段尚未创建 HTTP 适配器或服务器。如果您的测试需要 `httpAdapter`，您应该使用 `createNestApplication()` 方法来创建应用程序实例，或者重构您的项目，在初始化依赖图时避免这种依赖。

好了，让我们分解这个例子：

我们将运行中的应用的引用保存在 `app` 变量中，以便我们可以使用它来模拟 HTTP 请求。

我们使用 Supertest 的 `request()` 函数来模拟 HTTP 测试。我们希望这些 HTTP 请求路由到我们运行的 Nest 应用，因此我们将 Nest 底层的 HTTP 监听器的引用传递给 `request()` 函数（而 Nest 又可能由 Express 平台提供）。因此有了 `request(app.getHttpServer())` 的构造。对 `request()` 的调用给我们一个包装好的 HTTP 服务器，现在连接到 Nest 应用，它暴露了模拟实际 HTTP 请求的方法。例如，使用 `request(...).get('/cats')` 将向 Nest 应用发起一个请求，该请求与通过网络传入的 **实际** HTTP 请求 `get '/cats'` 相同。

在这个例子中，我们还提供了 `CatsService` 的替代（测试替身）实现，它只返回一个我们可以测试的硬编码值。使用 `overrideProvider()` 来提供这样的替代实现。类似地，Nest 提供了分别使用 `overrideModule()`、`overrideGuard()`、`overrideInterceptor()`、`overrideFilter()` 和 `overridePipe()` 方法来覆盖模块、守卫、拦截器、过滤器和管道的方法。

每个覆盖方法（除了 `overrideModule()`）都返回一个具有 3 种不同方法的对象，这些方法镜像了[自定义提供者](https://docs.nestjs.com/fundamentals/custom-providers)中描述的那些方法：

- `useClass`：您提供一个类，该类将被实例化以提供覆盖对象（提供者、守卫等）的实例。
- `useValue`：您提供一个实例来覆盖对象。
- `useFactory`：您提供一个返回实例的函数，该实例将覆盖对象。

另一方面，`overrideModule()` 返回一个具有 `useModule()` 方法的对象，您可以使用该方法提供一个模块来覆盖原始模块，如下所示：

```typescript
const moduleRef = await Test.createTestingModule({
  imports: [AppModule],
})
  .overrideModule(CatsModule)
  .useModule(AlternateCatsModule)
  .compile();
```

每种覆盖方法类型依次返回 `TestingModule` 实例，因此可以与[流式风格](https://en.wikipedia.org/wiki/Fluent_interface)中的其他方法链式调用。您应该在此类链的末尾使用 `compile()`，以导致 Nest 实例化并初始化模块。

此外，有时您可能希望提供一个自定义记录器，例如在运行测试时（例如，在 CI 服务器上）。使用 `setLogger()` 方法并传递一个满足 `LoggerService` 接口的对象，以指示 `TestModuleBuilder` 在测试期间如何记录日志（默认情况下，只有 "error" 日志会被记录到控制台）。

编译后的模块有几个有用的方法，如下表所述：

<table>
  <tr>
    <td>
      <code>createNestApplication()</code>
    </td>
    <td>
      基于给定模块创建并返回一个 Nest 应用程序（<code>INestApplication</code> 实例）。
      注意，您必须使用 <code>init()</code> 方法手动初始化应用程序。
    </td>
  </tr>
  <tr>
    <td>
      <code>createNestMicroservice()</code>
    </td>
    <td>
      基于给定模块创建并返回一个 Nest 微服务（<code>INestMicroservice</code> 实例）。
    </td>
  </tr>
  <tr>
    <td>
      <code>get()</code>
    </td>
    <td>
      检索应用程序上下文中可用的控制器或提供者（包括守卫、过滤器等）的静态实例。继承自<a href="/fundamentals/module-ref">模块引用</a>类。
    </td>
  </tr>
  <tr>
     <td>
      <code>resolve()</code>
    </td>
    <td>
      检索应用程序上下文中可用的控制器或提供者（包括守卫、过滤器等）的动态创建的作用域实例（请求或瞬态）。继承自<a href="/fundamentals/module-ref">模块引用</a>类。
    </td>
  </tr>
  <tr>
    <td>
      <code>select()</code>
    </td>
    <td>
      导航模块的依赖图；可用于从所选模块中检索特定实例（与 <code>get()</code> 方法中的严格模式（<code>strict: true</code>）一起使用）。
    </td>
  </tr>
</table>

> info **提示** 将您的 e2e 测试文件放在 `test` 目录中。测试文件应具有 `.e2e-spec` 后缀。

#### 覆盖全局注册的增强器

如果您有一个全局注册的守卫（或管道、拦截器、过滤器），您需要采取一些额外的步骤来覆盖该增强器。回顾一下原始的注册看起来像这样：

```typescript
providers: [
  {
    provide: APP_GUARD,
    useClass: JwtAuthGuard,
  },
],
```

这是通过 `APP_*` 令牌将守卫注册为“多”提供者。为了能够在这里替换 `JwtAuthGuard`，注册需要使用此槽位中的现有提供者：

```typescript
providers: [
  {
    provide: APP_GUARD,
    useExisting: JwtAuthGuard,
    // ^^^^^^^^ 注意使用 'useExisting' 而不是 'useClass'
  },
  JwtAuthGuard,
],
```

> info **提示** 将 `useClass` 更改为 `useExisting`，以引用已注册的提供者，而不是让 Nest 在令牌背后实例化它。

现在 `JwtAuthGuard` 对 Nest 可见，作为一个常规提供者，可以在创建 `TestingModule` 时被覆盖：

```typescript
const moduleRef = await Test.createTestingModule({
  imports: [AppModule],
})
  .overrideProvider(JwtAuthGuard)
  .useClass(MockAuthGuard)
  .compile();
```

现在您所有的测试将在每个请求上使用 `MockAuthGuard`。

#### 测试请求作用域的实例

[请求作用域](/fundamentals/injection-scopes)的提供者为每个传入的**请求**唯一创建。实例在请求处理完成后被垃圾回收。这带来了一个问题，因为我们无法访问为被测试请求专门生成的依赖注入子树。

我们知道（基于上面的部分）`resolve()` 方法可用于检索动态实例化的类。而且，如<a href="https://docs.nestjs.com/fundamentals/module-ref#resolving-scoped-providers">这里</a>所述，我们知道我们可以传递一个唯一的上下文标识符来控制 DI 容器子树的生命周期。我们如何在测试上下文中利用这一点？

策略是事先生成一个上下文标识符，并强制 Nest 使用这个特定的 ID 为所有传入请求创建一个子树。通过这种方式，我们将能够检索为被测试请求创建的实例。

为了实现这一点，在 `ContextIdFactory` 上使用 `jest.spyOn()`：

```typescript
const contextId = ContextIdFactory.create();
jest
  .spyOn(ContextIdFactory, 'getByRequest')
  .mockImplementation(() => contextId);
```

现在我们可以使用 `contextId` 来访问为任何后续请求生成的单个 DI 容器子树。

```typescript
catsService = await moduleRef.resolve(CatsService, contextId);
```