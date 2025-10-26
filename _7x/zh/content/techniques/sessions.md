### Session

**HTTP sessions** 提供了一种在多个请求之间存储用户信息的方法，这对于 [MVC](/techniques/mvc) 应用程序特别有用。

#### 与 Express 一起使用（默认）

首先安装 [所需的包](https://github.com/expressjs/session)（以及 TypeScript 用户所需的类型定义）：

```shell
$ npm i express-session
$ npm i -D @types/express-session
```

安装完成后，将 `express-session` 中间件应用为全局中间件（例如，在您的 `main.ts` 文件中）。

```typescript
import * as session from 'express-session';
// somewhere in your initialization file
app.use(
  session({
    secret: 'my-secret',
    resave: false,
    saveUninitialized: false,
  }),
);
```

> warning **注意** 默认的服务器端会话存储特意未设计用于生产环境。在大多数情况下，它会泄漏内存，无法扩展到单个进程之外，并且仅用于调试和开发。请在 [官方仓库](https://github.com/expressjs/session) 中阅读更多信息。

`secret` 用于签署会话 ID cookie。这可以是一个字符串（用于单个密钥）或一个包含多个密钥的数组。如果提供了密钥数组，则仅第一个元素将用于签署会话 ID cookie，而在验证请求中的签名时将考虑所有元素。密钥本身不应易于被人解析，最好是一组随机字符。

启用 `resave` 选项会强制将会话重新保存到会话存储中，即使在请求期间会话从未被修改过。默认值为 `true`，但使用默认值已被弃用，因为默认值将来会更改。

同样，启用 `saveUninitialized` 选项会强制将"未初始化"的会话保存到存储中。当会话是新的但未被修改时，它是未初始化的。选择 `false` 对于实现登录会话、减少服务器存储使用量或遵守设置 cookie 前需要获得许可的法律非常有用。选择 `false` 也有助于解决客户端在没有会话的情况下发出多个并行请求时的竞争条件问题（[来源](https://github.com/expressjs/session#saveuninitialized)）。

您可以向 `session` 中间件传递其他几个选项，有关它们的更多信息，请参阅 [API 文档](https://github.com/expressjs/session#options)。

> info **提示** 请注意，`secure: true` 是一个推荐的选项。但是，它要求网站启用了 https，即 HTTPS 对于安全 cookie 是必需的。如果设置了 secure，并且您通过 HTTP 访问您的站点，则不会设置 cookie。如果您的 node.js 位于代理后面并使用 `secure: true`，则需要在 express 中设置 `"trust proxy"`。

完成此设置后，您现在可以在路由处理程序中设置和读取会话值，如下所示：

```typescript
@Get()
findAll(@Req() request: Request) {
  request.session.visits = request.session.visits ? request.session.visits + 1 : 1;
}
```

> info **提示** `@Req()` 装饰器是从 `@nestjs/common` 导入的，而 `Request` 是从 `express` 包导入的。

或者，您可以使用 `@Session()` 装饰器从请求中提取会话对象，如下所示：

```typescript
@Get()
findAll(@Session() session: Record<string, any>) {
  session.visits = session.visits ? session.visits + 1 : 1;
}
```

> info **提示** `@Session()` 装饰器是从 `@nestjs/common` 包导入的。

#### 与 Fastify 一起使用

首先安装所需的包：

```shell
$ npm i @fastify/secure-session
```

安装完成后，注册 `fastify-secure-session` 插件：

```typescript
import secureSession from '@fastify/secure-session';

// somewhere in your initialization file
const app = await NestFactory.create<NestFastifyApplication>(
  AppModule,
  new FastifyAdapter(),
);
await app.register(secureSession, {
  secret: 'averylogphrasebiggerthanthirtytwochars',
  salt: 'mq9hDxBVDbspDR6n',
});
```

> info **提示** 您也可以预生成密钥（[查看说明](https://github.com/fastify/fastify-secure-session)）或使用 [密钥轮换](https://github.com/fastify/fastify-secure-session#using-keys-with-key-rotation)。

有关可用选项的更多信息，请参阅 [官方仓库](https://github.com/fastify/fastify-secure-session)。

完成此设置后，您现在可以在路由处理程序中设置和读取会话值，如下所示：

```typescript
@Get()
findAll(@Req() request: FastifyRequest) {
  const visits = request.session.get('visits');
  request.session.set('visits', visits ? visits + 1 : 1);
}
```

或者，您可以使用 `@Session()` 装饰器从请求中提取会话对象，如下所示：

```typescript
@Get()
findAll(@Session() session: secureSession.Session) {
  const visits = session.get('visits');
  session.set('visits', visits ? visits + 1 : 1);
}
```

> info **提示** `@Session()` 装饰器是从 `@nestjs/common` 导入的，而 `secureSession.Session` 是从 `@fastify/secure-session` 包导入的（导入语句：`import * as secureSession from '@fastify/secure-session'`）。