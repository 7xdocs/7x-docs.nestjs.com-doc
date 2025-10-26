### 速率限制

保护应用程序免受暴力攻击的一种常用技术是**速率限制**。要开始使用，你需要安装 `@nestjs/throttler` 包。

```bash
$ npm i --save @nestjs/throttler
```

安装完成后，可以像任何其他 Nest 包一样，使用 `forRoot` 或 `forRootAsync` 方法配置 `ThrottlerModule`。

```typescript
@@filename(app.module)
@Module({
  imports: [
    ThrottlerModule.forRoot([{
      ttl: 60000,
      limit: 10,
    }]),
  ],
})
export class AppModule {}
```

上述代码将为应用程序中受保护的路由设置全局选项：`ttl`（生存时间，以毫秒为单位）和 `limit`（在 ttl 时间范围内允许的最大请求数）。

一旦模块被导入，你可以选择如何绑定 `ThrottlerGuard`。任何在 [守卫](https://docs.nestjs.com/guards) 部分提到的绑定方式都是可以的。例如，如果你想全局绑定该守卫，可以通过在任何模块中添加此提供程序来实现：

```typescript
{
  provide: APP_GUARD,
  useClass: ThrottlerGuard
}
```

#### 多个限流器定义

有时你可能希望设置多个限流定义，例如：一秒内不超过 3 次调用，10 秒内不超过 20 次调用，一分钟内不超过 100 次调用。为此，你可以在数组中设置具有命名选项的定义，这些选项稍后可以在 `@SkipThrottle()` 和 `@Throttle()` 装饰器中引用，以再次更改选项。

```typescript
@@filename(app.module)
@Module({
  imports: [
    ThrottlerModule.forRoot([
      {
        name: 'short',
        ttl: 1000,
        limit: 3,
      },
      {
        name: 'medium',
        ttl: 10000,
        limit: 20
      },
      {
        name: 'long',
        ttl: 60000,
        limit: 100
      }
    ]),
  ],
})
export class AppModule {}
```

#### 自定义

有时你可能希望将守卫绑定到控制器或全局，但希望对一个或多个端点禁用速率限制。为此，你可以使用 `@SkipThrottle()` 装饰器，为整个类或单个路由取消限流器。`@SkipThrottle()` 装饰器也可以接受一个字符串键和布尔值的对象，用于在你想排除控制器中的*大部分*路由但不是所有路由的情况下，并且如果你有多个限流器集合，可以按每个限流器集进行配置。如果你不传递对象，则默认使用 `{{ '{' }} default: true {{ '}' }}`

```typescript
@SkipThrottle()
@Controller('users')
export class UsersController {}
```

这个 `@SkipThrottle()` 装饰器可用于跳过路由或类，或者用于在已跳过的类中取消对某个路由的跳过。

```typescript
@SkipThrottle()
@Controller('users')
export class UsersController {
  // 此路由应用速率限制。
  @SkipThrottle({ default: false })
  dontSkip() {
    return '列表用户工作正常，并应用了速率限制。';
  }
  // 此路由将跳过速率限制。
  doSkip() {
    return '列表用户工作正常，未应用速率限制。';
  }
}
```

还有一个 `@Throttle()` 装饰器，可用于覆盖全局模块中设置的 `limit` 和 `ttl`，以提供更严格或更宽松的安全选项。此装饰器也可以用在类或函数上。从版本 5 开始，装饰器接受一个对象，其中字符串与限流器集的名称相关，以及一个具有 limit 和 ttl 键和整数值的对象，类似于传递给根模块的选项。如果你在原始选项中没有设置名称，请使用字符串 `default`。你需要这样配置它：

```typescript
// 覆盖速率限制和持续时间的默认配置。
@Throttle({ default: { limit: 3, ttl: 60000 } })
@Get()
findAll() {
  return "列表用户工作正常，并使用了自定义速率限制。";
}
```

#### 代理

如果你的应用程序运行在代理服务器后面，配置 HTTP 适配器以信任代理至关重要。你可以参考 [Express](http://expressjs.com/en/guide/behind-proxies.html) 和 [Fastify](https://www.fastify.io/docs/latest/Reference/Server/#trustproxy) 的特定 HTTP 适配器选项来启用 `trust proxy` 设置。

以下示例演示了如何为 Express 适配器启用 `trust proxy`：

```typescript
@@filename(main.ts)
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { NestExpressApplication } from '@nestjs/platform-express';

async function bootstrap() {
  const app = await NestFactory.create<NestExpressApplication>(AppModule);
  app.set('trust proxy', 'loopback'); // 信任来自环回地址的请求
  await app.listen(3000);
}

bootstrap();
@@switch
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { NestExpressApplication } from '@nestjs/platform-express';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.set('trust proxy', 'loopback'); // 信任来自环回地址的请求
  await app.listen(3000);
}

bootstrap();
```

启用 `trust proxy` 允许你从 `X-Forwarded-For` 头中检索原始 IP 地址。你还可以通过重写 `getTracker()` 方法来自定义应用程序的行为，以从此头中提取 IP 地址，而不是依赖 `req.ip`。以下示例演示了如何为 Express 和 Fastify 实现这一点：

```typescript
@@filename(throttler-behind-proxy.guard)
import { ThrottlerGuard } from '@nestjs/throttler';
import { Injectable } from '@nestjs/common';

@Injectable()
export class ThrottlerBehindProxyGuard extends ThrottlerGuard {
  protected async getTracker(req: Record<string, any>): Promise<string> {
    return req.ips.length ? req.ips[0] : req.ip; // 根据你自己的需要个性化 IP 提取
  }
}
```

> info **提示** 你可以在 [这里](https://expressjs.com/en/api.html#req.ips) 找到 express 的 `req` 请求对象的 API，在 [这里](https://www.fastify.io/docs/latest/Reference/Request/) 找到 fastify 的。

#### WebSockets

该模块可以与 websockets 一起工作，但需要一些类扩展。你可以扩展 `ThrottlerGuard` 并重写 `handleRequest` 方法，如下所示：

```typescript
@Injectable()
export class WsThrottlerGuard extends ThrottlerGuard {
  async handleRequest(requestProps: ThrottlerRequest): Promise<boolean> {
    const {
      context,
      limit,
      ttl,
      throttler,
      blockDuration,
      getTracker,
      generateKey,
    } = requestProps;

    const client = context.switchToWs().getClient();
    const tracker = client._socket.remoteAddress;
    const key = generateKey(context, tracker, throttler.name);
    const { totalHits, timeToExpire, isBlocked, timeToBlockExpire } =
      await this.storageService.increment(
        key,
        ttl,
        limit,
        blockDuration,
        throttler.name,
      );

    const getThrottlerSuffix = (name: string) =>
      name === 'default' ? '' : `-${name}`;

    // 当用户达到限制时抛出错误。
    if (isBlocked) {
      await this.throwThrottlingException(context, {
        limit,
        ttl,
        key,
        tracker,
        totalHits,
        timeToExpire,
        isBlocked,
        timeToBlockExpire,
      });
    }

    return true;
  }
}
```

> info **提示** 如果你使用 ws，需要用 `conn` 替换 `_socket`

在使用 WebSockets 时，有几点需要记住：

- 守卫不能通过 `APP_GUARD` 或 `app.useGlobalGuards()` 注册
- 当达到限制时，Nest 会发出一个 `exception` 事件，因此请确保有一个监听器准备好处理此事件

> info **提示** 如果你使用 `@nestjs/platform-ws` 包，你可以使用 `client._socket.remoteAddress`。

#### GraphQL

`ThrottlerGuard` 也可以用于处理 GraphQL 请求。同样，守卫可以被扩展，但这次将重写 `getRequestResponse` 方法

```typescript
@Injectable()
export class GqlThrottlerGuard extends ThrottlerGuard {
  getRequestResponse(context: ExecutionContext) {
    const gqlCtx = GqlExecutionContext.create(context);
    const ctx = gqlCtx.getContext();
    return { req: ctx.req, res: ctx.res };
  }
}
```

#### 配置

以下选项对于传递给 `ThrottlerModule` 选项数组的对象是有效的：

<table>
  <tr>
    <td><code>name</code></td>
    <td>用于内部跟踪正在使用的限流器集的名称。如果未传递，则默认为 `default`</td>
  </tr>
  <tr>
    <td><code>ttl</code></td>
    <td>每个请求在存储中存活的毫秒数</td>
  </tr>
  <tr>
    <td><code>limit</code></td>
    <td>在 TTL 限制内的最大请求数</td>
  </tr>
  <tr>
    <td><code>blockDuration</code></td>
    <td>请求将被阻止的毫秒数</td>
  </tr>
  <tr>
    <td><code>ignoreUserAgents</code></td>
    <td>在限制请求时要忽略的用户代理的正则表达式数组</td>
  </tr>
  <tr>
    <td><code>skipIf</code></td>
    <td>一个接收 <code>ExecutionContext</code> 并返回 <code>boolean</code> 的函数，用于短路限流器逻辑。类似于 <code>@SkipThrottler()</code>，但基于请求</td>
  </tr>
</table>

如果你需要设置存储，或者希望更全局地使用上述某些选项，应用于每个限流器集，你可以通过 `throttlers` 选项键传递上述选项，并使用下表

<table>
  <tr>
    <td><code>storage</code></td>
    <td>用于跟踪限制的自定义存储服务。<a href="/security/rate-limiting#storages">参见此处。</a></td>
  </tr>
  <tr>
    <td><code>ignoreUserAgents</code></td>
    <td>在限制请求时要忽略的用户代理的正则表达式数组</td>
  </tr>
  <tr>
    <td><code>skipIf</code></td>
    <td>一个接收 <code>ExecutionContext</code> 并返回 <code>boolean</code> 的函数，用于短路限流器逻辑。类似于 <code>@SkipThrottler()</code>，但基于请求</td>
  </tr>
  <tr>
    <td><code>throttlers</code></td>
    <td>使用上表定义的限流器集数组</td>
  </tr>
  <tr>
    <td><code>errorMessage</code></td>
    <td>一个 <code>string</code> 或一个接收 <code>ExecutionContext</code> 和 <code>ThrottlerLimitDetail</code> 并返回 <code>string</code> 的函数，用于覆盖默认的限流器错误消息</td>
  </tr>
  <tr>
    <td><code>getTracker</code></td>
    <td>一个接收 <code>Request</code> 并返回 <code>string</code> 的函数，用于覆盖 <code>getTracker</code> 方法的默认逻辑</td>
  </tr>
  <tr>
    <td><code>generateKey</code></td>
    <td>一个接收 <code>ExecutionContext</code>、跟踪器 <code>string</code> 和限流器名称（作为 <code>string</code>）并返回 <code>string</code> 的函数，用于覆盖最终键，该键将用于存储速率限制值。这覆盖了 <code>generateKey</code> 方法的默认逻辑</td>
  </tr>
</table>

#### 异步配置

你可能希望异步获取速率限制配置，而不是同步。你可以使用 `forRootAsync()` 方法，它允许依赖注入和 `async` 方法。

一种方法是使用工厂函数：

```typescript
@Module({
  imports: [
    ThrottlerModule.forRootAsync({
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: (config: ConfigService) => [
        {
          ttl: config.get('THROTTLE_TTL'),
          limit: config.get('THROTTLE_LIMIT'),
        },
      ],
    }),
  ],
})
export class AppModule {}
```

你也可以使用 `useClass` 语法：

```typescript
@Module({
  imports: [
    ThrottlerModule.forRootAsync({
      imports: [ConfigModule],
      useClass: ThrottlerConfigService,
    }),
  ],
})
export class AppModule {}
```

只要 `ThrottlerConfigService` 实现了 `ThrottlerOptionsFactory` 接口，这是可行的。

#### 存储

内置存储是一个内存缓存，它跟踪发出的请求，直到它们通过了全局选项设置的 TTL。你可以将自己的存储选项放入 `ThrottlerModule` 的 `storage` 选项中，只要该类实现了 `ThrottlerStorage` 接口。

对于分布式服务器，你可以使用社区提供的 [Redis](https://github.com/jmcdo29/nest-lab/tree/main/packages/throttler-storage-redis) 存储提供程序，以拥有单一的事实来源。

> info **注意** `ThrottlerStorage` 可以从 `@nestjs/throttler` 导入。

#### 时间助手

有几个辅助方法可以使时间设置更易读，如果你更喜欢使用它们而不是直接定义的话。`@nestjs/throttler` 导出了五个不同的助手：`seconds`、`minutes`、`hours`、`days` 和 `weeks`。要使用它们，只需调用 `seconds(5)` 或任何其他助手，将返回正确的毫秒数。

#### 迁移指南

对于大多数人来说，将你的选项包装在一个数组中就足够了。

如果你使用自定义存储，你应该将你的 `ttl` 和 `limit` 包装在一个数组中，并将其分配给选项对象的 `throttlers` 属性。

任何 `@ThrottleSkip()` 现在应该接受一个具有 `string: boolean` 属性的对象。字符串是限流器的名称。如果你没有名称，请传递字符串 `'default'`，因为这是底层默认使用的名称。

任何 `@Throttle()` 装饰器现在也应该接受一个具有字符串键的对象，这些键与限流器上下文的名称相关（同样，如果没有名称，则使用 `'default'`），以及具有 `limit` 和 `ttl` 键的对象的值。

> Warning **重要** `ttl` 现在以**毫秒**为单位。如果你希望为了可读性而将 ttl 保留为秒，请使用此包中的 `seconds` 助手。它只是将 ttl 乘以 1000 以转换为毫秒。

更多信息，请参阅 [变更日志](https://github.com/nestjs/throttler/blob/master/CHANGELOG.md#500)