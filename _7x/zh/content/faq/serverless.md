### 无服务器架构（Serverless）

无服务器计算是一种云计算执行模型，在这种模型中，云服务提供商会按需分配机器资源，代表客户管理服务器。当应用程序未在使用时，不会为该应用程序分配任何计算资源。定价基于应用程序实际消耗的资源量（[来源](https://en.wikipedia.org/wiki/Serverless_computing)）。

借助**无服务器架构**，你可以纯粹专注于应用程序代码中的各个函数。像AWS Lambda、Google Cloud Functions和Microsoft Azure Functions这样的服务会处理所有物理硬件、虚拟机操作系统和Web服务器软件的管理工作。

> info **提示** 本章不涉及无服务器函数的优缺点，也不深入探讨任何云服务提供商的具体细节。

#### 冷启动（Cold start）

冷启动是指你的代码在一段时间内首次执行。根据你使用的云服务提供商不同，冷启动可能包含多个不同的操作，从下载代码、启动运行时到最终执行代码。
这个过程会增加**显著的延迟**，具体取决于多个因素，比如编程语言、应用程序所需的包数量等。

冷启动很重要，虽然有些因素超出了我们的控制范围，但我们仍然可以做很多事情来尽可能缩短冷启动时间。

虽然你可能认为Nest是一个为复杂的企业级应用程序设计的成熟框架，但它也**适用于许多“更简单”的应用程序**（或脚本）。例如，通过使用[独立应用程序](/standalone-applications)功能，你可以在简单的工作进程、CRON任务、CLI或无服务器函数中利用Nest的依赖注入（DI）系统。

#### 基准测试（Benchmarks）

为了更好地了解在无服务器函数环境中使用Nest或其他知名库（如`express`）的成本，让我们比较一下Node运行时执行以下脚本所需的时间：

```typescript
// #1 Express
import * as express from 'express';

async function bootstrap() {
  const app = express();
  app.get('/', (req, res) => res.send('Hello world!'));
  await new Promise<void>((resolve) => app.listen(3000, resolve));
}
bootstrap();

// #2 Nest (with @nestjs/platform-express)
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule, { logger: ['error'] });
  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();

// #3 Nest as a Standalone application (no HTTP server)
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { AppService } from './app.service';

async function bootstrap() {
  const app = await NestFactory.createApplicationContext(AppModule, {
    logger: ['error'],
  });
  console.log(app.get(AppService).getHello());
}
bootstrap();

// #4 Raw Node.js script
async function bootstrap() {
  console.log('Hello world!');
}
bootstrap();
```

对于所有这些脚本，我们使用了`tsc`（TypeScript）编译器，因此代码保持未捆绑状态（未使用`webpack`）。

| 类型 | 时间 |
| ------------------------------------ | ----------------- |
| Express                              | 0.0079s (7.9ms)   |
| Nest 搭配 `@nestjs/platform-express` | 0.1974s (197.4ms) |
| Nest（独立应用程序）        | 0.1117s (111.7ms) |
| 原生Node.js脚本                   | 0.0071s (7.1ms)   |

> info **注意** 测试机器：MacBook Pro 2014年中期款，2.5 GHz四核Intel Core i7，16 GB 1600 MHz DDR3，SSD。

现在，让我们重复所有基准测试，但这次使用`webpack`（如果你安装了[Nest CLI](/cli/overview)，可以运行`nest build --webpack`）将我们的应用程序捆绑到一个单独的可执行JavaScript文件中。
不过，我们不会使用Nest CLI附带的默认`webpack`配置，而是确保将所有依赖项（`node_modules`）捆绑在一起，如下所示：

```javascript
module.exports = (options, webpack) => {
  const lazyImports = [
    '@nestjs/microservices/microservices-module',
    '@nestjs/websockets/socket-module',
  ];

  return {
    ...options,
    externals: [],
    plugins: [
      ...options.plugins,
      new webpack.IgnorePlugin({
        checkResource(resource) {
          if (lazyImports.includes(resource)) {
            try {
              require.resolve(resource);
            } catch (err) {
              return true;
            }
          }
          return false;
        },
      }),
    ],
  };
};
```

> info **提示** 要指示Nest CLI使用此配置，请在项目的根目录中创建一个新的`webpack.config.js`文件。

使用此配置，我们得到了以下结果：

| 类型 | 时间 |
| ------------------------------------ | ---------------- |
| Express                              | 0.0068s (6.8ms)  |
| Nest 搭配 `@nestjs/platform-express` | 0.0815s (81.5ms) |
| Nest（独立应用程序）        | 0.0319s (31.9ms) |
| 原生Node.js脚本                   | 0.0066s (6.6ms)  |

> info **注意** 测试机器：MacBook Pro 2014年中期款，2.5 GHz四核Intel Core i7，16 GB 1600 MHz DDR3，SSD。

> info **提示** 你可以通过应用额外的代码压缩和优化技术（使用`webpack`插件等）进一步优化。

如你所见，编译方式（以及是否捆绑代码）至关重要，对整体启动时间有显著影响。使用`webpack`，你可以将独立Nest应用程序（带有一个模块、控制器和服务的初始项目）的启动时间平均降至约32ms，对于常规的基于express的NestJS HTTP应用程序，启动时间可降至约81.5ms。

对于更复杂的Nest应用程序，例如，有10个资源（通过`$ nest g resource` schematic生成 = 10个模块、10个控制器、10个服务、20个DTO类、50个HTTP端点 + `AppModule`），在MacBook Pro 2014年中期款（2.5 GHz四核Intel Core i7，16 GB 1600 MHz DDR3，SSD）上的整体启动时间约为0.1298s（129.8ms）。将单体应用程序作为无服务器函数运行通常没有太大意义，因此将此基准测试更多地视为应用程序增长时启动时间可能如何增加的示例。

#### 运行时优化（Runtime optimizations）

到目前为止，我们已经介绍了编译时优化。这些与你定义提供者和加载Nest模块的方式无关，而随着应用程序变大，这一点起着至关重要的作用。

例如，想象有一个数据库连接被定义为[异步提供者](/fundamentals/async-providers)。异步提供者旨在延迟应用程序启动，直到一个或多个异步任务完成。
这意味着，如果你的无服务器函数在启动时平均需要2秒来连接数据库，那么你的端点（在冷启动且应用程序尚未运行的情况下）至少需要额外两秒才能发送响应（因为它必须等待连接建立）。

如你所见，在**无服务器环境**中，你构建提供者的方式有所不同，因为启动时间很重要。
另一个很好的例子是，如果你使用Redis进行缓存，但只在某些情况下使用。或许，在这种情况下，你不应该将Redis连接定义为异步提供者，因为这会减慢启动时间，即使在这个特定的函数调用中不需要它。

此外，有时你可以使用`LazyModuleLoader`类懒加载整个模块，如[本章](/fundamentals/lazy-loading-modules)所述。缓存也是一个很好的例子。
想象你的应用程序有一个`CacheModule`，它内部连接到Redis，并且导出`CacheService`以与Redis存储交互。如果你不需要它用于所有可能的函数调用，
你可以按需懒加载它。这样，对于所有不需要缓存的调用，（冷启动时的）启动时间会更快。

```typescript
if (request.method === RequestMethod[RequestMethod.GET]) {
  const { CacheModule } = await import('./cache.module');
  const moduleRef = await this.lazyModuleLoader.load(() => CacheModule);

  const { CacheService } = await import('./cache.service');
  const cacheService = moduleRef.get(CacheService);

  return cacheService.get(ENDPOINT_KEY);
}
```

另一个很好的例子是webhook或工作进程，它们可能根据某些特定条件（例如，输入参数）执行不同的操作。
在这种情况下，你可以在路由处理程序中指定一个条件，为特定的函数调用懒加载相应的模块，并懒加载所有其他模块。

```typescript
if (workerType === WorkerType.A) {
  const { WorkerAModule } = await import('./worker-a.module');
  const moduleRef = await this.lazyModuleLoader.load(() => WorkerAModule);
  // ...
} else if (workerType === WorkerType.B) {
  const { WorkerBModule } = await import('./worker-b.module');
  const moduleRef = await this.lazyModuleLoader.load(() => WorkerBModule);
  // ...
}
```

#### 示例集成（Example integration）

你的应用程序入口文件（通常是`main.ts`文件）的外观**取决于多个因素**，因此**没有一个适用于所有场景的单一模板**。
例如，启动无服务器函数所需的初始化文件因云服务提供商（AWS、Azure、GCP等）而异。
此外，根据你是想运行一个具有多个路由/端点的典型HTTP应用程序，还是只提供一个单一路由（或执行特定代码部分），
你的应用程序代码会有所不同（例如，对于“每个函数一个端点”的方法，你可以使用`NestFactory.createApplicationContext`，而不是启动HTTP服务器、设置中间件等）。

仅为了说明，我们将Nest（使用`@nestjs/platform-express`，因此启动完整功能的HTTP路由器）与[Serverless](https://www.serverless.com/)框架（在本例中，目标是AWS Lambda）集成。如前所述，你的代码会因所选的云服务提供商以及许多其他因素而有所不同。

首先，让我们安装所需的包：

```bash
$ npm i @codegenie/serverless-express aws-lambda
$ npm i -D @types/aws-lambda serverless-offline
```

> info **提示** 为了加快开发周期，我们安装`serverless-offline`插件，它可以模拟AWS λ和API Gateway。

安装过程完成后，让我们创建`serverless.yml`文件来配置Serverless框架：

```yaml
service: serverless-example

plugins:
  - serverless-offline

provider:
  name: aws
  runtime: nodejs14.x

functions:
  main:
    handler: dist/main.handler
    events:
      - http:
          method: ANY
          path: /
      - http:
          method: ANY
          path: '{proxy+}'
```

> info **提示** 要了解更多关于Serverless框架的信息，请访问[官方文档](https://www.serverless.com/framework/docs/)。

完成上述设置后，我们可以导航到`main.ts`文件，并使用所需的样板代码更新我们的启动代码：

```typescript
import { NestFactory } from '@nestjs/core';
import serverlessExpress from '@codegenie/serverless-express';
import { Callback, Context, Handler } from 'aws-lambda';
import { AppModule } from './app.module';

let server: Handler;

async function bootstrap(): Promise<Handler> {
  const app = await NestFactory.create(AppModule);
  await app.init();

  const expressApp = app.getHttpAdapter().getInstance();
  return serverlessExpress({ app: expressApp });
}

export const handler: Handler = async (
  event: any,
  context: Context,
  callback: Callback,
) => {
  server = server ?? (await bootstrap());
  return server(event, context, callback);
};
```

> info **提示** 对于创建多个无服务器函数并在它们之间共享公共模块，我们建议使用[CLI Monorepo模式](/cli/monorepo#monorepo-mode)。

> warning **警告** 如果你使用`@nestjs/swagger`包，要使其在无服务器函数环境中正常工作，需要执行一些额外步骤。查看此[线程](https://github.com/nestjs/swagger/issues/199)了解更多信息。

接下来，打开`tsconfig.json`文件，确保启用`esModuleInterop`选项，以使`@codegenie/serverless-express`包正确加载。

```json
{
  "compilerOptions": {
    ...
    "esModuleInterop": true
  }
}
```

现在，我们可以构建应用程序（使用`nest build`或`tsc`），并使用`serverless` CLI在本地启动lambda函数：

```bash
$ npm run build
$ npx serverless offline
```

应用程序运行后，打开浏览器并导航到`http://localhost:3000/dev/[ANY_ROUTE]`（其中`[ANY_ROUTE]`是你应用程序中注册的任何端点）。

在上面的部分中，我们已经展示了使用`webpack`并捆绑应用程序可以对整体启动时间产生显著影响。
但是，要使其与我们的示例一起工作，你必须在`webpack.config.js`文件中添加一些额外的配置。通常，
为了确保我们的`handler`函数被正确识别，我们必须将`output.libraryTarget`属性更改为`commonjs2`。

```javascript
return {
  ...options,
  externals: [],
  output: {
    ...options.output,
    libraryTarget: 'commonjs2',
  },
  // ... 其余配置
};
```

完成此设置后，你可以使用`$ nest build --webpack`编译函数代码（然后使用`$ npx serverless offline`进行测试）。

还建议（但**不是必须**，因为它会减慢构建过程）安装`terser-webpack-plugin`包，并覆盖其配置，以便在压缩生产构建时保持类名不变。否则，在应用程序中使用`class-validator`时可能会导致不正确的行为。

```javascript
const TerserPlugin = require('terser-webpack-plugin');

return {
  ...options,
  externals: [],
  optimization: {
    minimizer: [
      new TerserPlugin({
        terserOptions: {
          keep_classnames: true,
        },
      }),
    ],
  },
  output: {
    ...options.output,
    libraryTarget: 'commonjs2',
  },
  // ... 其余配置
};
```

#### 使用独立应用程序功能（Using standalone application feature）

或者，如果你想让你的函数非常轻量，并且不需要任何与HTTP相关的功能（路由，以及守卫、拦截器、管道等），
你可以只使用`NestFactory.createApplicationContext`（如前所述），而不是运行整个HTTP服务器（以及底层的`express`），如下所示：

```typescript
@@filename(main)
import { HttpStatus } from '@nestjs/common';
import { NestFactory } from '@nestjs/core';
import { Callback, Context, Handler } from 'aws-lambda';
import { AppModule } from './app.module';
import { AppService } from './app.service';

export const handler: Handler = async (
  event: any,
  context: Context,
  callback: Callback,
) => {
  const appContext = await NestFactory.createApplicationContext(AppModule);
  const appService = appContext.get(AppService);

  return {
    body: appService.getHello(),
    statusCode: HttpStatus.OK,
  };
};
```

> info **提示** 请注意，`NestFactory.createApplicationContext`不会用增强器（守卫、拦截器等）包装控制器方法。要实现这一点，你必须使用`NestFactory.create`方法。

你也可以将`event`对象传递给例如`EventsService`提供者，该提供者可以处理它并返回相应的值（取决于输入值和你的业务逻辑）。

```typescript
export const handler: Handler = async (
  event: any,
  context: Context,
  callback: Callback,
) => {
  const appContext = await NestFactory.createApplicationContext(AppModule);
  const eventsService = appContext.get(EventsService);
  return eventsService.process(event);
};
```