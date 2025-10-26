### 缓存

缓存是一种强大且直接的**技术**，用于提升应用程序的性能。它作为临时存储层，允许快速访问频繁使用的数据，减少重复获取或计算相同信息的需求。这带来了更快的响应时间和更高的整体效率。

#### 安装

要开始使用 Nest 中的缓存，你需要安装 `@nestjs/cache-manager` 包以及 `cache-manager` 包。

```bash
$ npm install @nestjs/cache-manager cache-manager
```

默认情况下，所有内容都存储在内存中；由于 `cache-manager` 底层使用 [Keyv](https://keyv.org/docs/)，你可以通过安装适当的包轻松切换到更高级的存储解决方案，例如 Redis。我们将在后面更详细地介绍这一点。

#### 内存缓存

要在应用程序中启用缓存，导入 `CacheModule` 并使用 `register()` 方法进行配置：

```typescript
import { Module } from '@nestjs/common';
import { CacheModule } from '@nestjs/cache-manager';
import { AppController } from './app.controller';

@Module({
  imports: [CacheModule.register()],
  controllers: [AppController],
})
export class AppModule {}
```

此设置使用默认设置初始化内存缓存，允许你立即开始缓存数据。

#### 与缓存存储交互

要与缓存管理器实例交互，请使用 `CACHE_MANAGER` 令牌将其注入到你的类中，如下所示：

```typescript
constructor(@Inject(CACHE_MANAGER) private cacheManager: Cache) {}
```

> info **提示** `Cache` 类是从 `cache-manager` 导入的，而 `CACHE_MANAGER` 令牌是从 `@nestjs/cache-manager` 包导入的。

`Cache` 实例（来自 `cache-manager` 包）上的 `get` 方法用于从缓存中检索项目。如果缓存中不存在该条目，将返回 `null`。

```typescript
const value = await this.cacheManager.get('key');
```

要向缓存添加一个条目，使用 `set` 方法：

```typescript
await this.cacheManager.set('key', 'value');
```

> warning **注意** 内存缓存存储只能存储[结构化克隆算法](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Structured_clone_algorithm#javascript_types)支持的类型值。

缓存的默认过期时间是 5 秒。

你可以为此特定键手动指定 TTL（以秒为单位的过期时间），如下所示：

```typescript
await this.cacheManager.set('key', 'value', 1000);
```

要禁用缓存过期，请将 `ttl` 配置属性设置为 `0`：

```typescript
await this.cacheManager.set('key', 'value', 0);
```

要从缓存中移除一个条目，使用 `del` 方法：

```typescript
await this.cacheManager.del('key');
```

要清空整个缓存，使用 `reset` 方法：

```typescript
await this.cacheManager.reset();
```

#### 自动缓存响应

> warning **警告** 在 [GraphQL](/graphql/quick-start) 应用程序中，拦截器会为每个字段解析器单独执行。因此，`CacheModule`（使用拦截器来缓存响应）将无法正常工作。

要启用自动缓存响应，只需在你想要缓存数据的地方绑定 `CacheInterceptor`。

```typescript
@Controller()
@UseInterceptors(CacheInterceptor)
export class AppController {
  @Get()
  findAll(): string[] {
    return [];
  }
}
```

> warning **警告** 只有 `GET` 端点会被缓存。此外，注入原生响应对象（`@Res()`）的 HTTP 服务器路由无法使用缓存拦截器。有关更多详细信息，请参阅 <a href="https://docs.nestjs.com/interceptors#response-mapping">响应映射</a>。

为了减少所需的样板代码，你可以将 `CacheInterceptor` 全局绑定到所有端点：

```typescript
import { Module } from '@nestjs/common';
import { CacheModule, CacheInterceptor } from '@nestjs/cache-manager';
import { AppController } from './app.controller';
import { APP_INTERCEPTOR } from '@nestjs/core';

@Module({
  imports: [CacheModule.register()],
  controllers: [AppController],
  providers: [
    {
      provide: APP_INTERCEPTOR,
      useClass: CacheInterceptor,
    },
  ],
})
export class AppModule {}
```

#### 自定义缓存

所有缓存数据都有其自己的过期时间（[TTL](https://en.wikipedia.org/wiki/Time_to_live)）。要自定义默认值，请将选项对象传递给 `register()` 方法。

```typescript
CacheModule.register({
  ttl: 5, // 秒
  max: 10, // 缓存中的最大条目数
});
```

#### 全局使用模块

当你想在其他模块中使用 `CacheModule` 时，需要导入它（就像任何标准 Nest 模块一样）。或者，通过将选项对象的 `isGlobal` 属性设置为 `true` 将其声明为[全局模块](https://docs.nestjs.com/modules#global-modules)，如下所示。在这种情况下，一旦在根模块（例如 `AppModule`）中加载了 `CacheModule`，你就不需要在其他模块中再次导入它。

```typescript
CacheModule.register({
  isGlobal: true,
});
```

#### 全局缓存覆盖

当启用全局缓存时，缓存条目存储在基于路由路径自动生成的 `CacheKey` 下。你可以基于每个方法覆盖某些缓存设置（`@CacheKey()` 和 `@CacheTTL()`），从而为单个控制器方法允许自定义缓存策略。这在[使用不同的缓存存储](https://docs.nestjs.com/techniques/caching#different-stores)时可能最相关。

你可以在每个控制器上应用 `@CacheTTL()` 装饰器，为整个控制器设置缓存 TTL。在同时定义了控制器级别和方法级别缓存 TTL 设置的情况下，方法级别指定的缓存 TTL 设置将优先于控制器级别的设置。

```typescript
@Controller()
@CacheTTL(50)
export class AppController {
  @CacheKey('custom_key')
  @CacheTTL(20)
  findAll(): string[] {
    return [];
  }
}
```

> info **提示** `@CacheKey()` 和 `@CacheTTL()` 装饰器是从 `@nestjs/cache-manager` 包导入的。

`@CacheKey()` 装饰器可以单独使用，也可以与相应的 `@CacheTTL()` 装饰器一起使用，反之亦然。可以选择仅覆盖 `@CacheKey()` 或仅覆盖 `@CacheTTL()`。未使用装饰器覆盖的设置将使用全局注册的默认值（参见[自定义缓存](https://docs.nestjs.com/techniques/caching#customize-caching)）。

#### WebSockets 和微服务

你也可以将 `CacheInterceptor` 应用于 WebSocket 订阅者以及微服务的模式（无论使用何种传输方法）。

```typescript
@@filename()
@CacheKey('events')
@UseInterceptors(CacheInterceptor)
@SubscribeMessage('events')
handleEvent(client: Client, data: string[]): Observable<string[]> {
  return [];
}
@@switch
@CacheKey('events')
@UseInterceptors(CacheInterceptor)
@SubscribeMessage('events')
handleEvent(client, data) {
  return [];
}
```

但是，需要额外的 `@CacheKey()` 装饰器来指定用于后续存储和检索缓存数据的键。此外，请注意你**不应缓存所有内容**。执行某些业务操作而不仅仅是查询数据的操作永远不应该被缓存。

此外，你可以使用 `@CacheTTL()` 装饰器指定缓存过期时间（TTL），这将覆盖全局默认的 TTL 值。

```typescript
@@filename()
@CacheTTL(10)
@UseInterceptors(CacheInterceptor)
@SubscribeMessage('events')
handleEvent(client: Client, data: string[]): Observable<string[]> {
  return [];
}
@@switch
@CacheTTL(10)
@UseInterceptors(CacheInterceptor)
@SubscribeMessage('events')
handleEvent(client, data) {
  return [];
}
```

> info **提示** `@CacheTTL()` 装饰器可以单独使用，也可以与相应的 `@CacheKey()` 装饰器一起使用。

#### 调整跟踪

默认情况下，Nest 使用请求 URL（在 HTTP 应用中）或缓存键（在 WebSocket 和微服务应用中，通过 `@CacheKey()` 装饰器设置）来将缓存记录与你的端点关联起来。然而，有时你可能希望基于不同的因素来设置跟踪，例如使用 HTTP 头（例如 `Authorization` 来正确识别 `profile` 端点）。

为了实现这一点，创建一个 `CacheInterceptor` 的子类并重写 `trackBy()` 方法。

```typescript
@Injectable()
class HttpCacheInterceptor extends CacheInterceptor {
  trackBy(context: ExecutionContext): string | undefined {
    return 'key';
  }
}
```

#### 使用替代缓存存储

切换到不同的缓存存储很简单。首先，安装适当的包。例如，要使用 Redis，请安装 `@keyv/redis` 包：

```bash
$ npm install @keyv/redis
```

完成此操作后，你可以注册具有多个存储的 `CacheModule`，如下所示：

```typescript
import { Module } from '@nestjs/common';
import { CacheModule, CacheStore } from '@nestjs/cache-manager';
import { AppController } from './app.controller';
import KeyvRedis from '@keyv/redis';
import { Keyv } from 'keyv';
import { CacheableMemory } from 'cacheable';

@Module({
  imports: [
    CacheModule.registerAsync({
      useFactory: async () => {
        return {
          stores: [
            new Keyv({
              store: new CacheableMemory({ ttl: 60000, lruSize: 5000 }),
            }),
            new KeyvRedis('redis://localhost:6379'),
          ],
        };
      },
    }),
  ],
  controllers: [AppController],
})
export class AppModule {}
```

在这个例子中，我们注册了两个存储：`CacheableMemory` 和 `KeyvRedis`。`CacheableMemory` 存储是一个简单的内存存储，而 `KeyvRedis` 是一个 Redis 存储。`stores` 数组用于指定你想要使用的存储。数组中的第一个存储是默认存储，其余的是后备存储。

查看 [Keyv 文档](https://keyv.org/docs/) 以获取有关可用存储的更多信息。

#### 异步配置

你可能希望异步传递模块选项，而不是在编译时静态传递。在这种情况下，使用 `registerAsync()` 方法，它提供了几种处理异步配置的方式。

一种方法是使用工厂函数：

```typescript
CacheModule.registerAsync({
  useFactory: () => ({
    ttl: 5,
  }),
});
```

我们的工厂的行为与所有其他异步模块工厂一样（它可以是 `async` 的，并且能够通过 `inject` 注入依赖项）。

```typescript
CacheModule.registerAsync({
  imports: [ConfigModule],
  useFactory: async (configService: ConfigService) => ({
    ttl: configService.get('CACHE_TTL'),
  }),
  inject: [ConfigService],
});
```

或者，你可以使用 `useClass` 方法：

```typescript
CacheModule.registerAsync({
  useClass: CacheConfigService,
});
```

上述构造将在 `CacheModule` 内部实例化 `CacheConfigService`，并使用它来获取选项对象。`CacheConfigService` 必须实现 `CacheOptionsFactory` 接口以提供配置选项：

```typescript
@Injectable()
class CacheConfigService implements CacheOptionsFactory {
  createCacheOptions(): CacheModuleOptions {
    return {
      ttl: 5,
    };
  }
}
```

如果你希望使用从其他模块导入的现有配置提供程序，请使用 `useExisting` 语法：

```typescript
CacheModule.registerAsync({
  imports: [ConfigModule],
  useExisting: ConfigService,
});
```

这与 `useClass` 的工作方式相同，但有一个关键区别——`CacheModule` 将查找已导入的模块以重用任何已创建的 `ConfigService`，而不是实例化自己的。

> info **提示** `CacheModule#register`、`CacheModule#registerAsync` 和 `CacheOptionsFactory` 有一个可选的泛型（类型参数）来缩小特定于存储的配置选项，使其类型安全。

你也可以将所谓的 `extraProviders` 传递给 `registerAsync()` 方法。这些提供程序将与模块提供程序合并。

```typescript
CacheModule.registerAsync({
  imports: [ConfigModule],
  useClass: ConfigService,
  extraProviders: [MyAdditionalProvider],
});
```

当你希望向工厂函数或类构造函数提供额外的依赖项时，这很有用。

#### 示例

一个可运行的示例可在[这里](https://github.com/nestjs/nest/tree/master/sample/20-cache)找到。