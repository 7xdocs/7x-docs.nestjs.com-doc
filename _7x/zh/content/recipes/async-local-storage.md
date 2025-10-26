### Async Local Storage（异步本地存储）

`AsyncLocalStorage` 是一个 [Node.js API](https://nodejs.org/api/async_context.html#async_context_class_asynclocalstorage)（基于 `async_hooks` API），它提供了一种在应用程序中传播本地状态的方式，而无需将其作为函数参数显式传递。它类似于其他语言中的线程本地存储。

Async Local Storage 的主要思想是，我们可以使用 `AsyncLocalStorage#run` 调用来_包装_某个函数调用。在包装调用内调用的所有代码都可以访问相同的 `store`，该 store 对于每个调用链将是唯一的。

在 NestJS 的上下文中，这意味着如果我们在请求的生命周期内找到一个可以包装请求其余代码的位置，我们将能够访问和修改仅对该请求可见的状态，这可以作为 REQUEST 作用域提供者及其某些限制的替代方案。

或者，我们可以使用 ALS 来传播系统的仅一部分上下文（例如 _transaction_ 对象），而无需在服务之间显式传递它，这可以增加隔离性和封装性。

#### 自定义实现

NestJS 本身没有为 `AsyncLocalStorage` 提供任何内置抽象，所以让我们逐步了解如何自己为最简单的 HTTP 情况实现它，以便更好地理解整个概念：

> info **提示** 如需现成的[专用包](recipes/async-local-storage#nestjs-cls)，请继续阅读下文。

1. 首先，在某个共享源文件中创建一个新的 `AsyncLocalStorage` 实例。由于我们使用 NestJS，让我们也将其转换为带有自定义提供者的模块。

```ts
@@filename(als.module)
@Module({
  providers: [
    {
      provide: AsyncLocalStorage,
      useValue: new AsyncLocalStorage(),
    },
  ],
  exports: [AsyncLocalStorage],
})
export class AlsModule {}
```
>  info **提示** `AsyncLocalStorage` 从 `async_hooks` 导入。

2. 我们只关注 HTTP，所以让我们使用一个中间件来用 `AsyncLocalStorage#run` 包装 `next` 函数。由于中间件是请求首先到达的地方，这将使 `store` 在所有增强器和系统的其余部分中可用。

```ts
@@filename(app.module)
@Module({
  imports: [AlsModule]
  providers: [CatService],
  controllers: [CatController],
})
export class AppModule implements NestModule {
  constructor(
    // 在模块构造函数中注入 AsyncLocalStorage，
    private readonly als: AsyncLocalStorage
  ) {}

  configure(consumer: MiddlewareConsumer) {
    // 绑定中间件，
    consumer
      .apply((req, res, next) => {
        // 根据请求，用一些默认值填充 store，
        const store = {
          userId: req.headers['x-user-id'],
        };
        // 并将 "next" 函数作为回调与 store 一起传递给 "als.run" 方法。
        this.als.run(store, () => next());
      })
      // 并将其注册到所有路由（对于 Fastify，使用 '(.*)'）
      .forRoutes('*');
  }
}
@@switch
@Module({
  imports: [AlsModule]
  providers: [CatService],
  controllers: [CatController],
})
@Dependencies(AsyncLocalStorage)
export class AppModule {
  constructor(als) {
    // 在模块构造函数中注入 AsyncLocalStorage，
    this.als = als
  }

  configure(consumer) {
    // 绑定中间件，
    consumer
      .apply((req, res, next) => {
        // 根据请求，用一些默认值填充 store，
        const store = {
          userId: req.headers['x-user-id'],
        };
        // 并将 "next" 函数作为回调与 store 一起传递给 "als.run" 方法。
        this.als.run(store, () => next());
      })
      // 并将其注册到所有路由（对于 Fastify，使用 '(.*)'）
      .forRoutes('*');
  }
}
```

3. 现在，在请求生命周期的任何地方，我们都可以访问本地 store 实例。

```ts
@@filename(cat.service)
@Injectable()
export class CatService {
  constructor(
    // 我们可以注入提供的 ALS 实例。
    private readonly als: AsyncLocalStorage,
    private readonly catRepository: CatRepository,
  ) {}

  getCatForUser() {
    // "getStore" 方法将始终返回与给定请求关联的 store 实例。
    const userId = this.als.getStore()["userId"] as number;
    return this.catRepository.getForUser(userId);
  }
}
@@switch
@Injectable()
@Dependencies(AsyncLocalStorage, CatRepository)
export class CatService {
  constructor(als, catRepository) {
    // 我们可以注入提供的 ALS 实例。
    this.als = als
    this.catRepository = catRepository
  }

  getCatForUser() {
    // "getStore" 方法将始终返回与给定请求关联的 store 实例。
    const userId = this.als.getStore()["userId"] as number;
    return this.catRepository.getForUser(userId);
  }
}
```

4. 就是这样。现在我们有一种共享请求相关状态的方式，而无需注入整个 `REQUEST` 对象。

> warning **警告** 请注意，虽然该技术对许多用例很有用，但它本质上模糊了代码流程（创建了隐式上下文），因此请负责任地使用它，尤其要避免创建上下文的“[上帝对象](https://en.wikipedia.org/wiki/God_object)”。

### NestJS CLS

[nestjs-cls](https://github.com/Papooch/nestjs-cls) 包在使用纯 `AsyncLocalStorage` 的基础上提供了多个开发者体验改进（`CLS` 是术语 _continuation-local storage_ 的缩写）。它将实现抽象到一个 `ClsModule` 中，该模块提供了为不同传输方式（不仅仅是 HTTP）初始化 `store` 的各种方法，以及强类型支持。

然后可以使用可注入的 `ClsService` 访问 store，或者通过使用[代理提供者](https://www.npmjs.com/package/nestjs-cls#proxy-providers)将其完全从业务逻辑中抽象出来。

> info **提示** `nestjs-cls` 是第三方包，不由 NestJS 核心团队管理。请在该库的[相应仓库](https://github.com/Papooch/nestjs-cls/issues)中报告发现的任何问题。

#### 安装

除了对 `@nestjs` 库的 peer 依赖外，它只使用内置的 Node.js API。像安装其他包一样安装它。

```bash
npm i nestjs-cls
```

#### 用法

使用 `nestjs-cls` 可以实现与[上文](recipes/async-local-storage#custom-implementation)描述的类似功能，如下所示：

1. 在根模块中导入 `ClsModule`。

```ts
@@filename(app.module)
@Module({
  imports: [
    // 注册 ClsModule，
    ClsModule.forRoot({
      middleware: {
        // 自动为所有路由挂载 ClsMiddleware
        mount: true,
        // 并使用 setup 方法提供默认的 store 值。
        setup: (cls, req) => {
          cls.set('userId', req.headers['x-user-id']);
        },
      },
    }),
  ],
  providers: [CatService],
  controllers: [CatController],
})
export class AppModule {}
```

2. 然后可以使用 `ClsService` 来访问 store 值。

```ts
@@filename(cat.service)
@Injectable()
export class CatService {
  constructor(
    // 我们可以注入提供的 ClsService 实例，
    private readonly cls: ClsService,
    private readonly catRepository: CatRepository,
  ) {}

  getCatForUser() {
    // 并使用 "get" 方法检索任何存储的值。
    const userId = this.cls.get('userId');
    return this.catRepository.getForUser(userId);
  }
}
@@switch
@Injectable()
@Dependencies(AsyncLocalStorage, CatRepository)
export class CatService {
  constructor(cls, catRepository) {
    // 我们可以注入提供的 ClsService 实例，
    this.cls = cls
    this.catRepository = catRepository
  }

  getCatForUser() {
    // 并使用 "get" 方法检索任何存储的值。
    const userId = this.cls.get('userId');
    return this.catRepository.getForUser(userId);
  }
}
```

3. 为了获得由 `ClsService` 管理的 store 值的强类型（以及获得字符串键的自动建议），我们可以在注入时使用可选的类型参数 `ClsService<MyClsStore>`。

```ts
export interface MyClsStore extends ClsStore {
  userId: number;
}
```

> info **提示** 还可以让包自动生成一个请求 ID，然后使用 `cls.getId()` 访问它，或者使用 `cls.get(CLS_REQ)` 获取整个请求对象。

#### 测试

由于 `ClsService` 只是另一个可注入的提供者，它可以在单元测试中被完全模拟。

然而，在某些集成测试中，我们可能仍然希望使用真实的 `ClsService` 实现。在这种情况下，我们需要使用 `ClsService#run` 或 `ClsService#runWith` 调用来包装具有上下文感知的代码片段。

```ts
describe('CatService', () => {
  let service: CatService
  let cls: ClsService
  const mockCatRepository = createMock<CatRepository>()

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      // 像平时一样设置测试模块的大部分。
      providers: [
        CatService,
        {
          provide: CatRepository
          useValue: mockCatRepository
        }
      ],
      imports: [
        // 导入静态版本的 ClsModule，它只提供 ClsService，但不以任何方式设置 store。
        ClsModule
      ],
    }).compile()

    service = module.get(CatService)

    // 同时获取 ClsService 供后续使用。
    cls = module.get(ClsService)
  })

  describe('getCatForUser', () => {
    it('retrieves cat based on user id', async () => {
      const expectedUserId = 42
      mockCatRepository.getForUser.mockImplementationOnce(
        (id) => ({ userId: id })
      )

      // 将测试调用包装在 `runWith` 方法中，
      // 在该方法中我们可以传递手工制作的 store 值。
      const cat = await cls.runWith(
        { userId: expectedUserId },
        () => service.getCatForUser()
      )

      expect(cat.userId).toEqual(expectedUserId)
    })
  })
})
```

#### 更多信息

访问 [NestJS CLS GitHub 页面](https://github.com/Papooch/nestjs-cls) 获取完整的 API 文档和更多代码示例。