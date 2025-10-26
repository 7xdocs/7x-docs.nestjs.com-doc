### Read-Eval-Print-Loop (REPL)

REPL 是一个简单的交互式环境，它接收单个用户输入，执行它们，并将结果返回给用户。
REPL 功能允许您从终端直接检查您的依赖图并在您的提供者（和控制器）上调用方法。

#### 用法

要在 REPL 模式下运行您的 NestJS 应用程序，请创建一个新的 `repl.ts` 文件（与现有的 `main.ts` 文件放在一起），并在其中添加以下代码：

```typescript
@@filename(repl)
import { repl } from '@nestjs/core';
import { AppModule } from './src/app.module';

async function bootstrap() {
  await repl(AppModule);
}
bootstrap();
@@switch
import { repl } from '@nestjs/core';
import { AppModule } from './src/app.module';

async function bootstrap() {
  await repl(AppModule);
}
bootstrap();
```

现在在您的终端中，使用以下命令启动 REPL：

```bash
$ npm run start -- --entryFile repl
```

> info **提示** `repl` 返回一个 [Node.js REPL 服务器](https://nodejs.org/api/repl.html) 对象。

一旦它启动并运行，您应该在控制台中看到以下消息：

```bash
LOG [NestFactory] Starting Nest application...
LOG [InstanceLoader] AppModule dependencies initialized
LOG REPL initialized
```

现在您可以开始与您的依赖图进行交互。例如，您可以获取一个 `AppService`（我们这里使用入门项目作为示例）并调用 `getHello()` 方法：

```typescript
> get(AppService).getHello()
'Hello World!'
```

您可以从终端内执行任何 JavaScript 代码，例如，将 `AppController` 的一个实例分配给一个局部变量，并使用 `await` 调用异步方法：

```typescript
> appController = get(AppController)
AppController { appService: AppService {} }
> await appController.getHello()
'Hello World!'
```

要显示给定提供者或控制器上所有可用的公共方法，请使用 `methods()` 函数，如下所示：

```typescript
> methods(AppController)

Methods:
 ◻ getHello
```

要将所有已注册的模块及其控制器和提供者以列表形式打印出来，请使用 `debug()`。

```typescript
> debug()

AppModule:
 - controllers:
  ◻ AppController
 - providers:
  ◻ AppService
```

快速演示：

<figure><img src="/assets/repl.gif" alt="REPL 示例" /></figure>

您可以在下面的部分中找到有关现有的、预定义的原生方法的更多信息。

#### 原生函数

内置的 NestJS REPL 附带了一些在您启动 REPL 时全局可用的原生函数。您可以调用 `help()` 来列出它们。

如果您不记得某个函数的签名（即：预期的参数和返回类型），您可以调用 `<function_name>.help`。
例如：

```text
> $.help
检索可注入对象或控制器的实例，否则抛出异常。
接口: $(token: InjectionToken) => any
```

> info **提示** 这些函数接口是使用 [TypeScript 函数类型表达式语法](https://www.typescriptlang.org/docs/handbook/2/functions.html#function-type-expressions) 编写的。

| 函数        | 描述                                                                                             | 签名                                                                  |
| ----------- | ------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------- |
| `debug`     | 将所有已注册的模块及其控制器和提供者以列表形式打印出来。                                         | `debug(moduleCls?: ClassRef \| string) => void`                       |
| `get` 或 `$` | 检索可注入对象或控制器的实例，否则抛出异常。                                                     | `get(token: InjectionToken) => any`                                   |
| `methods`   | 显示给定提供者或控制器上所有可用的公共方法。                                                     | `methods(token: ClassRef \| string) => void`                          |
| `resolve`   | 解析可注入对象或控制器的临时或请求作用域的实例，否则抛出异常。                                   | `resolve(token: InjectionToken, contextId: any) => Promise<any>`      |
| `select`    | 允许在模块树中导航，例如，从选中的模块中拉取特定的实例。                                         | `select(token: DynamicModule \| ClassRef) => INestApplicationContext` |

#### 监视模式

在开发过程中，以监视模式运行 REPL 非常有用，可以自动反映所有代码更改：

```bash
$ npm run start -- --watch --entryFile repl
```

这有一个缺陷，REPL 的命令历史记录在每次重新加载后都会被丢弃，这可能很麻烦。
幸运的是，有一个非常简单的解决方案。像这样修改您的 `bootstrap` 函数：

```typescript
async function bootstrap() {
  const replServer = await repl(AppModule);
  replServer.setupHistory(".nestjs_repl_history", (err) => {
    if (err) {
      console.error(err);
    }
  });
}
```

现在历史记录在运行/重新加载之间得以保留。