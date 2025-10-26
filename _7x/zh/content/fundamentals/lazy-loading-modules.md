### 延迟加载模块

默认情况下，模块是急切加载的，这意味着应用程序一旦加载，所有模块也会立即加载，无论它们是否立即需要。虽然这对于大多数应用程序来说没问题，但对于在**无服务器环境**中运行的应用程序/工作器来说，可能会成为一个瓶颈，因为启动延迟（"冷启动"）至关重要。

延迟加载可以通过仅加载特定无服务器函数调用所需的模块来减少引导时间。此外，一旦无服务器函数"变热"，您还可以异步加载其他模块，以进一步加快后续调用的引导时间（延迟模块注册）。

> info **提示** 如果您熟悉 **[Angular](https://angular.dev/)** 框架，可能之前见过"[延迟加载模块](https://angular.dev/guide/ngmodules/lazy-loading#lazy-loading-basics)"这个术语。请注意，该技术在 Nest 中**功能上是不同的**，因此请将其视为一个完全不同的功能，只是共享了类似的命名约定。

> warning **警告** 请注意，[生命周期钩子方法](https://docs.nestjs.com/fundamentals/lifecycle-events)在延迟加载的模块和服务中不会被调用。

#### 开始使用

要按需加载模块，Nest 提供了 `LazyModuleLoader` 类，可以以通常的方式注入到类中：

```typescript
@@filename(cats.service)
@Injectable()
export class CatsService {
  constructor(private lazyModuleLoader: LazyModuleLoader) {}
}
@@switch
@Injectable()
@Dependencies(LazyModuleLoader)
export class CatsService {
  constructor(lazyModuleLoader) {
    this.lazyModuleLoader = lazyModuleLoader;
  }
}
```

> info **提示** `LazyModuleLoader` 类是从 `@nestjs/core` 包中导入的。

或者，您可以从应用程序引导文件（`main.ts`）中获取 `LazyModuleLoader` 提供者的引用，如下所示：

```typescript
// "app" 代表一个 Nest 应用程序实例
const lazyModuleLoader = app.get(LazyModuleLoader);
```

有了这个之后，您现在可以使用以下结构加载任何模块：

```typescript
const { LazyModule } = await import('./lazy.module');
const moduleRef = await this.lazyModuleLoader.load(() => LazyModule);
```

> info **提示** "延迟加载"的模块在第一次调用 `LazyModuleLoader#load` 方法时会被**缓存**。这意味着，后续每次尝试加载 `LazyModule` 都会**非常快**，并返回一个缓存的实例，而不是再次加载该模块。
>
> ```bash
> 加载 "LazyModule" 尝试: 1
> 时间: 2.379ms
> 加载 "LazyModule" 尝试: 2
> 时间: 0.294ms
> 加载 "LazyModule" 尝试: 3
> 时间: 0.303ms
> ```
>
> 此外，"延迟加载"的模块与在应用程序引导时急切加载的模块以及之后在应用程序中注册的任何其他延迟模块共享相同的模块图。

其中 `lazy.module.ts` 是一个导出**常规 Nest 模块**的 TypeScript 文件（不需要额外的更改）。

`LazyModuleLoader#load` 方法返回 [模块引用](/fundamentals/module-ref)（`LazyModule` 的），它允许您浏览内部的提供者列表，并使用其注入令牌作为查找键来获取任何提供者的引用。

例如，假设我们有一个 `LazyModule`，其定义如下：

```typescript
@Module({
  providers: [LazyService],
  exports: [LazyService],
})
export class LazyModule {}
```

> info **提示** 延迟加载的模块不能注册为**全局模块**，因为这完全没有意义（因为它们是延迟注册的，按需的，此时所有静态注册的模块都已经被实例化）。同样，注册的**全局增强器**（守卫/拦截器等）也将无法正常工作。

有了这个，我们可以获取 `LazyService` 提供者的引用，如下所示：

```typescript
const { LazyModule } = await import('./lazy.module');
const moduleRef = await this.lazyModuleLoader.load(() => LazyModule);

const { LazyService } = await import('./lazy.service');
const lazyService = moduleRef.get(LazyService);
```

> warning **警告** 如果您使用 **Webpack**，请确保更新您的 `tsconfig.json` 文件 - 将 `compilerOptions.module` 设置为 `"esnext"`，并添加 `compilerOptions.moduleResolution` 属性，其值为 `"node"`：
>
> ```json
> {
>   "compilerOptions": {
>     "module": "esnext",
>     "moduleResolution": "node",
>     ...
>   }
> }
> ```
>
> 设置了这些选项后，您将能够利用[代码分割](https://webpack.js.org/guides/code-splitting/)功能。

#### 延迟加载控制器、网关和解析器

由于 Nest 中的控制器（或 GraphQL 应用程序中的解析器）代表一组路由/路径/主题（或查询/变更），因此您**不能**使用 `LazyModuleLoader` 类来延迟加载它们。

> error **警告** 在延迟加载模块内注册的控制器、[解析器](/graphql/resolvers)和[网关](/websockets/gateways)将无法按预期工作。同样，您也无法按需注册中间件函数（通过实现 `MiddlewareConsumer` 接口）。

例如，假设您正在构建一个基于 Fastify 驱动（使用 `@nestjs/platform-fastify` 包）的 REST API（HTTP 应用程序）。Fastify 不允许您在应用程序准备就绪/成功监听消息后注册路由。这意味着，即使我们分析了模块控制器中注册的路由映射，所有延迟加载的路由也将无法访问，因为在运行时无法注册它们。

同样，我们作为 `@nestjs/microservices` 包一部分提供的某些传输策略（包括 Kafka、gRPC 或 RabbitMQ）需要在建立连接之前订阅/监听特定的主题/通道。一旦您的应用程序开始监听消息，框架将无法订阅/监听新的主题。

最后，启用代码优先方法的 `@nestjs/graphql` 包会根据元数据动态生成 GraphQL 模式。这意味着，它要求所有类都预先加载完毕。否则，将无法创建正确、有效的模式。

#### 常见用例

最常见的情况是，当您的工作器/定时任务/Lambda 和无服务器函数/Webhook 必须根据输入参数（路由路径/日期/查询参数等）触发不同的服务（不同的逻辑）时，您会看到延迟加载的模块。另一方面，对于单体应用程序，延迟加载模块可能意义不大，因为启动时间相对无关紧要。