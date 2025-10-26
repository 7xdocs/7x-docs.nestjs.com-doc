### 生命周期事件

Nest 应用程序以及每个应用程序元素都有一个由 Nest 管理的生命周期。Nest 提供了**生命周期钩子**，可以查看关键生命周期事件，并在这些事件发生时采取行动（在模块、提供者或控制器上运行已注册的代码）。

#### 生命周期顺序

下图描绘了关键应用程序生命周期事件的顺序，从应用程序启动开始，直到节点进程退出。我们可以将整个生命周期分为三个阶段：**初始化**、**运行**和**终止**。利用这个生命周期，你可以规划模块和服务的适当初始化，管理活动连接，并在应用程序收到终止信号时优雅地关闭它。

<figure><img class="illustrative-image" src="/assets/lifecycle-events.png" /></figure>

#### 生命周期事件

生命周期事件发生在应用程序启动和关闭期间。Nest 在以下每个生命周期事件时，会在模块、提供者和控制器上调用已注册的生命周期钩子方法（**需要先启用关闭钩子**，如下所述）。如上图所示，Nest 还会调用适当的底层方法来开始监听连接和停止监听连接。

在下表中，仅当你显式调用 `app.init()` 或 `app.listen()` 时，才会触发 `onModuleInit` 和 `onApplicationBootstrap`。

在下表中，仅当你显式调用 `app.close()` 或者进程收到特殊的系统信号（例如 SIGTERM）并且你在应用程序引导时正确调用了 `enableShutdownHooks`（参见下面的**应用程序关闭**部分），才会触发 `onModuleDestroy`、`beforeApplicationShutdown` 和 `onApplicationShutdown`。

| 生命周期钩子方法               | 触发钩子方法调用的生命周期事件                                                                                                                                                                 |
| ------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `onModuleInit()`               | 在宿主模块的依赖项解析完成后调用。                                                                                                                                                             |
| `onApplicationBootstrap()`     | 在所有模块初始化之后，但在开始监听连接之前调用。                                                                                                                                               |
| `onModuleDestroy()`\*          | 在收到终止信号（例如 `SIGTERM`）后调用。                                                                                                                                                       |
| `beforeApplicationShutdown()`\* | 在所有 `onModuleDestroy()` 处理程序完成（Promise 已 resolved 或 rejected）后调用；<br />一旦完成（Promise 已 resolved 或 rejected），所有现有连接将被关闭（调用 `app.close()`）。 |
| `onApplicationShutdown()`\*    | 在连接关闭后调用（`app.close()` 已 resolved）。                                                                                                                                                |

\* 对于这些事件，如果你没有显式调用 `app.close()`，则必须选择加入才能使它们与系统信号（如 `SIGTERM`）一起工作。请参见下面的[应用程序关闭](fundamentals/lifecycle-events#application-shutdown)。

> warning **警告** 上面列出的生命周期钩子不会在**请求作用域**的类上触发。请求作用域的类不绑定到应用程序生命周期，其生命周期是不可预测的。它们专门为每个请求创建，并在响应发送后自动进行垃圾回收。

> info **提示** `onModuleInit()` 和 `onApplicationBootstrap()` 的执行顺序直接取决于模块导入的顺序，会等待前一个钩子完成。

#### 用法

每个生命周期钩子都由一个接口表示。从技术上讲，接口是可选的，因为它们在 TypeScript 编译后不存在。尽管如此，使用它们以获得强类型和编辑器工具的支持是一个好习惯。要注册一个生命周期钩子，请实现相应的接口。例如，要在特定类（例如控制器、提供者或模块）上注册一个在模块初始化期间调用的方法，请通过提供 `onModuleInit()` 方法来实现 `OnModuleInit` 接口，如下所示：

```typescript
@@filename()
import { Injectable, OnModuleInit } from '@nestjs/common';

@Injectable()
export class UsersService implements OnModuleInit {
  onModuleInit() {
    console.log(`The module has been initialized.`);
  }
}
@@switch
import { Injectable } from '@nestjs/common';

@Injectable()
export class UsersService {
  onModuleInit() {
    console.log(`The module has been initialized.`);
  }
}
```

#### 异步初始化

`OnModuleInit` 和 `OnApplicationBootstrap` 钩子都允许你延迟应用程序初始化过程（返回一个 `Promise` 或将方法标记为 `async` 并在方法体中 `await` 异步方法完成）。

```typescript
@@filename()
async onModuleInit(): Promise<void> {
  await this.fetch();
}
@@switch
async onModuleInit() {
  await this.fetch();
}
```

#### 应用程序关闭

`onModuleDestroy()`、`beforeApplicationShutdown()` 和 `onApplicationShutdown()` 钩子在终止阶段被调用（响应显式调用 `app.close()` 或在选择加入的情况下收到系统信号如 SIGTERM 时）。此功能通常与 [Kubernetes](https://kubernetes.io/) 一起用于管理容器的生命周期，或者被 [Heroku](https://www.heroku.com/) 用于 dynos 或类似服务。

关闭钩子监听器会消耗系统资源，因此默认情况下是禁用的。要使用关闭钩子，你**必须启用监听器**，通过调用 `enableShutdownHooks()`：

```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // 开始监听关闭钩子
  app.enableShutdownHooks();

  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

> warning **警告** 由于固有的平台限制，NestJS 在 Windows 上对应用程序关闭钩子的支持有限。你可以预期 `SIGINT` 会工作，`SIGBREAK` 也会，并且在某种程度上 `SIGHUP` 也会 - [阅读更多](https://nodejs.org/api/process.html#process_signal_events)。但是 `SIGTERM` 在 Windows 上永远无法工作，因为在任务管理器中终止进程是无条件的，"即应用程序无法检测或阻止它"。这里有一些来自 libuv 的[相关文档](https://docs.libuv.org/en/v1.x/signal.html)，以了解更多关于 `SIGINT`、`SIGBREAK` 和其他信号在 Windows 上如何处理的信息。另请参阅 Node.js 文档关于[进程信号事件](https://nodejs.org/api/process.html#process_signal_events)的部分。

> info **信息** `enableShutdownHooks` 通过启动监听器来消耗内存。在单个 Node 进程中运行多个 Nest 应用程序的情况下（例如，在使用 Jest 运行并行测试时），Node 可能会抱怨过多的监听器进程。因此，默认情况下未启用 `enableShutdownHooks`。在单个 Node 进程中运行多个实例时，请注意此情况。

当应用程序收到终止信号时，它将使用相应的信号作为第一个参数，按照上述顺序调用任何已注册的 `onModuleDestroy()`、`beforeApplicationShutdown()`，然后 `onApplicationShutdown()` 方法。如果注册的函数等待异步调用（返回一个 promise），Nest 将不会继续执行序列，直到该 promise 被 resolved 或 rejected。

```typescript
@@filename()
@Injectable()
class UsersService implements OnApplicationShutdown {
  onApplicationShutdown(signal: string) {
    console.log(signal); // 例如 "SIGINT"
  }
}
@@switch
@Injectable()
class UsersService implements OnApplicationShutdown {
  onApplicationShutdown(signal) {
    console.log(signal); // 例如 "SIGINT"
  }
}
```

> info **信息** 调用 `app.close()` 不会终止 Node 进程，只会触发 `onModuleDestroy()` 和 `onApplicationShutdown()` 钩子，因此如果有一些间隔、长时间运行的后台任务等，进程不会自动终止。