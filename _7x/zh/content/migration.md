### 迁移指南

本文提供了从 NestJS 版本 10 迁移到版本 11 的全面指南。要了解 v11 引入的新功能，请查看[这篇文章](#)。虽然此更新包含一些小的破坏性变更，但它们不太可能影响大多数用户。你可以在此处查看完整的破坏性变更列表[here](#)。

#### 升级包

虽然你可以手动升级你的包，但我们建议使用 [npm-check-updates (ncu)](https://npmjs.com/package/npm-check-updates) 以获得更简化的过程。

#### Express v5

经过多年的开发，Express v5 于 2024 年正式发布，并于 2025 年成为稳定版本。在 NestJS 11 中，Express v5 现在是框架内集成的默认版本。虽然此更新对大多数用户来说是无缝的，但务必注意 Express v5 引入了一些破坏性变更。有关详细指导，请参阅 [Express v5 迁移指南](https://expressjs.com/en/guide/migrating-5.html)。

Express v5 中最显著的更新之一是修订了路径路由匹配算法。以下是对路径字符串与传入请求匹配方式的变更：

- 通配符 `*` 必须有一个名称，与参数 `:` 的行为相匹配，使用 `/*splat` 或 `/{{ '{' }}*splat&#125;` 而不是 `/*`
- 不再支持可选字符 `?`，请改用花括号：`/:file{{ '{' }}.:ext&#125;`。
- 不支持正则表达式字符。
- 为避免升级过程中的混淆，保留了一些字符 `(()[]?+!)`，使用 `\` 对它们进行转义。
- 参数名称现在支持有效的 JavaScript 标识符，或者使用引号，如 `:"this"`。

也就是说，在 Express v4 中正常工作的路由在 Express v5 中可能无法工作。例如：

```typescript
@Get('users/*')
findAll() {
  // In NestJS 11, this will be automatically converted to a valid Express v5 route.
  // While it may still work, it's no longer advisable to use this wildcard syntax in Express v5.
  return 'This route should not work in Express v5';
}
```

要修复此问题，你可以更新路由以使用命名通配符：

```typescript
@Get('users/*splat')
findAll() {
  return 'This route will work in Express v5';
}
```

> warning **警告** 请注意，`*splat` 是一个命名通配符，它匹配任何不包含根路径的路径。如果你还需要匹配根路径（`/users`），你可以使用 `/users/{{ '{' }}*splat&#125;`，将通配符包裹在花括号中（可选组）。

类似地，如果你有一个在所有路由上运行的中间件，你可能需要更新路径以使用命名通配符：

```typescript
// In NestJS 11, this will be automatically converted to a valid Express v5 route.
// While it may still work, it's no longer advisable to use this wildcard syntax in Express v5.
forRoutes('*'); // <-- This should not work in Express v5
```

相反，你可以更新路径以使用命名通配符：

```typescript
forRoutes('{*splat}'); // <-- This will work in Express v5
```

请注意，`{{ '{' }}*splat&#125;` 是一个命名通配符，它匹配包括根路径在内的任何路径。外层的花括号使路径变为可选。

#### Fastify v5

Fastify v5 于 2024 年发布，现在是 NestJS 11 中集成的默认版本。此更新对大多数用户来说应该是无缝的；然而，Fastify v5 引入了一些破坏性变更，尽管这些不太可能影响大多数 NestJS 用户。更多详细信息，请参阅 [Fastify v5 迁移指南](https://fastify.dev/docs/v5.1.x/Guides/Migration-Guide-V5/)。

> info **提示** Fastify v5 中的路径匹配没有变化，因此你可以像以前一样继续使用通配符语法。行为保持不变，使用通配符（如 `*`）定义的路由仍将按预期工作。

#### 模块解析算法

从 NestJS 11 开始，模块解析算法得到了改进，以提高大多数应用程序的性能并减少内存使用。此变更不需要任何手动干预，但在某些边缘情况下，其行为可能与先前版本不同。

在 NestJS v10 及更早版本中，动态模块被分配了一个从模块的动态元数据生成的唯一不透明键。该键用于在模块注册表中识别模块。例如，如果你在多个模块中包含 `TypeOrmModule.forFeature([User])`，NestJS 将对模块进行去重，并在注册表中将它们视为单个模块节点。此过程称为节点去重。

随着 NestJS v11 的发布，我们不再为动态模块生成可预测的哈希值。现在，使用对象引用确定一个模块是否与另一个模块等效。要在多个模块之间共享相同的动态模块，只需将其分配给一个变量并在需要的地方导入它。这种新方法提供了更大的灵活性，并确保更有效地处理动态模块。

#### Reflector 类型推断

NestJS 11 引入了对 `Reflector` 类的几项改进，增强了其功能和对元数据值的类型推断。这些更新在使用元数据时提供了更直观和更健壮的体验。

1. 当只有一个元数据条目且 `value` 的类型为 `object` 时，`getAllAndMerge` 现在返回一个对象，而不是包含单个元素的数组。此变更提高了处理基于对象的元数据时的一致性。
2. `getAllAndOverride` 的返回类型已更新为 `T | undefined` 而不是 `T`。此更新更好地反映了可能找不到元数据的情况，并确保正确处理 undefined 情况。
3. `ReflectableDecorator` 的转换类型参数现在可以在所有方法中正确推断。

这些增强功能通过提供更好的类型安全性和对元数据的处理，改善了 NestJS 11 的整体开发者体验。

#### 生命周期钩子执行顺序

终止生命周期钩子现在按照其初始化对应钩子的相反顺序执行。也就是说，像 `OnModuleDestroy`、`BeforeApplicationShutdown` 和 `OnApplicationShutdown` 这样的钩子现在以相反的顺序执行。

设想以下场景：

```plaintext
// 其中 A、B 和 C 是模块，"->" 代表模块依赖关系。
A -> B -> C
```

在这种情况下，`OnModuleInit` 钩子按以下顺序执行：

```plaintext
C -> B -> A
```

而 `OnModuleDestroy` 钩子以相反的顺序执行：

```plaintext
A -> B -> C
```

> info **提示** 全局模块被视为依赖于所有其他模块。这意味着全局模块首先初始化，最后销毁。

#### 缓存模块

`CacheModule`（来自 `@nestjs/cache-manager` 包）已更新，以支持最新版本的 `cache-manager` 包。此更新带来了一些破坏性变更，包括迁移到 [Keyv](https://keyv.org/)，它通过存储适配器为多个后端存储提供了统一的键值存储接口。

先前版本和新版本之间的关键区别在于外部存储的配置。在先前版本中，要注册 Redis 存储，你可能会这样配置：

```ts
// 旧版本 - 不再受支持
CacheModule.registerAsync({
  useFactory: async () => {
    const store = await redisStore({
      socket: {
        host: 'localhost',
        port: 6379,
      },
    });

    return {
      store,
    };
  },
}),
```

在新版本中，你应该使用 `Keyv` 适配器来配置存储：

```ts
// 新版本 - 受支持
CacheModule.registerAsync({
  useFactory: async () => {
    return {
      stores: [
        new KeyvRedis('redis://localhost:6379'),
      ],
    };
  },
}),
```

其中 `KeyvRedis` 是从 `@keyv/redis` 包导入的。请参阅 [缓存文档](/techniques/caching) 以了解更多信息。

#### 配置模块

如果你使用来自 `@nestjs/config` 包的 `ConfigModule`，请注意 `@nestjs/config@4.0.0` 中引入的几个破坏性变更。最值得注意的是，`ConfigService#get` 方法读取配置变量的顺序已更新。新的顺序是：

- 内部配置（配置命名空间和自定义配置文件）
- 已验证的环境变量（如果启用了验证并提供了模式）
- `process.env` 对象

以前，已验证的环境变量和 `process.env` 对象首先被读取，这阻止了它们被内部配置覆盖。通过此次更新，内部配置现在将始终优先于环境变量。

此外，先前允许禁用 `process.env` 对象验证的 `ignoreEnvVars` 配置选项已被弃用。相反，请使用 `validatePredefined` 选项（设置为 `false` 以禁用对预定义环境变量的验证）。预定义环境变量是指在导入模块之前设置的 `process.env` 变量。例如，如果你使用 `PORT=3000 node main.js` 启动应用程序，则 `PORT` 变量被视为预定义的。但是，由 `ConfigModule` 从 `.env` 文件加载的变量不被归类为预定义的。

还引入了一个新的 `skipProcessEnv` 选项。此选项允许你完全阻止 `ConfigService#get` 方法访问 `process.env` 对象，当你想限制服务直接读取环境变量时，这会很有帮助。

#### 不再支持 Node.js v16

从 NestJS 11 开始，不再支持 Node.js v16，因为它已于 2023 年 9 月 11 日终止支持 (EOL)。NestJS 11 现在要求 **Node.js v20 或更高版本**。

为确保最佳体验，我们强烈建议使用最新的 Node.js LTS 版本。