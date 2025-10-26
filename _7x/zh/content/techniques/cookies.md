### Cookies

**HTTP cookie** 是存储在用户浏览器中的一小段数据。Cookie 被设计为一种可靠的机制，供网站记住状态信息。当用户再次访问该网站时，Cookie 会自动随请求一起发送。

#### 在 Express 中使用（默认）

首先安装[所需的包](https://github.com/expressjs/cookie-parser)（对于 TypeScript 用户，还需要安装其类型定义）：

```shell
$ npm i cookie-parser
$ npm i -D @types/cookie-parser
```

安装完成后，将 `cookie-parser` 中间件应用为全局中间件（例如，在您的 `main.ts` 文件中）。

```typescript
import * as cookieParser from 'cookie-parser';
// 在您的初始化文件中的某个位置
app.use(cookieParser());
```

您可以向 `cookieParser` 中间件传递几个选项：

- `secret` 一个字符串或数组，用于签署 cookie。这是可选的，如果未指定，则不会解析已签名的 cookie。如果提供了字符串，则将其用作密钥。如果提供了数组，则将尝试按顺序使用每个密钥来验证 cookie 的签名。
- `options` 一个对象，作为第二个选项传递给 `cookie.parse`。更多信息请参阅 [cookie](https://www.npmjs.org/package/cookie)。

该中间件将解析请求中的 `Cookie` 头，并将 cookie 数据暴露为属性 `req.cookies`，如果提供了密钥，则还会暴露为属性 `req.signedCookies`。这些属性是 cookie 名称到 cookie 值的键值对。

当提供了密钥时，此模块将验证任何已签名的 cookie 值，并将这些键值对从 `req.cookies` 移动到 `req.signedCookies`。已签名的 cookie 是其值带有 `s:` 前缀的 cookie。签名验证失败的已签名 cookie 将具有值 `false`，而不是被篡改的值。

完成此设置后，您现在可以在路由处理程序中读取 cookie，如下所示：

```typescript
@Get()
findAll(@Req() request: Request) {
  console.log(request.cookies); // 或 "request.cookies['cookieKey']"
  // 或 console.log(request.signedCookies);
}
```

> info **提示** `@Req()` 装饰器从 `@nestjs/common` 导入，而 `Request` 从 `express` 包导入。

要将 cookie 附加到传出的响应中，请使用 `Response#cookie()` 方法：

```typescript
@Get()
findAll(@Res({ passthrough: true }) response: Response) {
  response.cookie('key', 'value')
}
```

> warning **警告** 如果您希望将响应处理逻辑留给框架处理，请记住将 `passthrough` 选项设置为 `true`，如上所示。阅读更多 [这里](/controllers#library-specific-approach)。

> info **提示** `@Res()` 装饰器从 `@nestjs/common` 导入，而 `Response` 从 `express` 包导入。

#### 在 Fastify 中使用

首先安装所需的包：

```shell
$ npm i @fastify/cookie
```

安装完成后，注册 `@fastify/cookie` 插件：

```typescript
import fastifyCookie from '@fastify/cookie';

// 在您的初始化文件中的某个位置
const app = await NestFactory.create<NestFastifyApplication>(AppModule, new FastifyAdapter());
await app.register(fastifyCookie, {
  secret: 'my-secret', // 用于 cookie 签名
});
```

完成此设置后，您现在可以在路由处理程序中读取 cookie，如下所示：

```typescript
@Get()
findAll(@Req() request: FastifyRequest) {
  console.log(request.cookies); // 或 "request.cookies['cookieKey']"
}
```

> info **提示** `@Req()` 装饰器从 `@nestjs/common` 导入，而 `FastifyRequest` 从 `fastify` 包导入。

要将 cookie 附加到传出的响应中，请使用 `FastifyReply#setCookie()` 方法：

```typescript
@Get()
findAll(@Res({ passthrough: true }) response: FastifyReply) {
  response.setCookie('key', 'value')
}
```

要阅读有关 `FastifyReply#setCookie()` 方法的更多信息，请查看此[页面](https://github.com/fastify/fastify-cookie#sending)。

> warning **警告** 如果您希望将响应处理逻辑留给框架处理，请记住将 `passthrough` 选项设置为 `true`，如上所示。阅读更多 [这里](/controllers#library-specific-approach)。

> info **提示** `@Res()` 装饰器从 `@nestjs/common` 导入，而 `FastifyReply` 从 `fastify` 包导入。

#### 创建自定义装饰器（跨平台）

为了提供一种方便的、声明式的方法来访问传入的 cookie，我们可以创建一个[自定义装饰器](/custom-decorators)。

```typescript
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const Cookies = createParamDecorator((data: string, ctx: ExecutionContext) => {
  const request = ctx.switchToHttp().getRequest();
  return data ? request.cookies?.[data] : request.cookies;
});
```

`@Cookies()` 装饰器将从 `req.cookies` 对象中提取所有 cookie 或指定名称的 cookie，并用该值填充被装饰的参数。

完成此设置后，我们现在可以在路由处理程序签名中使用该装饰器，如下所示：

```typescript
@Get()
findAll(@Cookies('name') name: string) {}
```