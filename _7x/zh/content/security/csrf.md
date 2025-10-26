### CSRF 保护

跨站请求伪造（CSRF 或 XSRF）是一种攻击类型，其中**未经授权**的命令从受信任的用户发送到 Web 应用程序。为了帮助防止这种情况，您可以使用 [csrf-csrf](https://github.com/Psifi-Solutions/csrf-csrf) 包。

#### 与 Express 一起使用（默认）

首先安装所需的包：

```bash
$ npm i csrf-csrf
```

> warning **警告** 正如 [csrf-csrf 文档](https://github.com/Psifi-Solutions/csrf-csrf?tab=readme-ov-file#getting-started) 中所述，此中间件需要会话中间件或事先初始化的 `cookie-parser`。请参阅文档以获取更多详细信息。

安装完成后，将 `csrf-csrf` 中间件注册为全局中间件。

```typescript
import { doubleCsrf } from 'csrf-csrf';
// ...
// 在你的初始化文件中的某个位置
const {
  invalidCsrfTokenError, // 这纯粹是为了方便，如果你计划创建自己的中间件。
  generateToken, // 在你的路由中使用它来生成并提供 CSRF 哈希，以及一个令牌 cookie 和令牌。
  validateRequest, // 同样是为了方便，如果你计划创建自己的中间件。
  doubleCsrfProtection, // 这是默认的 CSRF 保护中间件。
} = doubleCsrf(doubleCsrfOptions);
app.use(doubleCsrfProtection);
```

#### 与 Fastify 一起使用

首先安装所需的包：

```bash
$ npm i --save @fastify/csrf-protection
```

安装完成后，注册 `@fastify/csrf-protection` 插件，如下所示：

```typescript
import fastifyCsrf from '@fastify/csrf-protection';
// ...
// 在注册某个存储插件后的初始化文件中的某个位置
await app.register(fastifyCsrf);
```

> warning **警告** 正如 `@fastify/csrf-protection` 文档 [此处](https://github.com/fastify/csrf-protection#usage) 所解释的，此插件需要先初始化一个存储插件。请参阅该文档以获取进一步的说明。