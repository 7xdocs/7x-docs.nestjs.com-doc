### 控制器

控制器负责处理传入的**请求**并向客户端返回**响应**。

<figure><img class="illustrative-image" src="/assets/Controllers_1.png" /></figure>

控制器的目的是接收应用程序的特定请求。**路由**机制控制哪个控制器接收哪些请求。通常，每个控制器有多个路由，不同的路由可以执行不同的操作。

为了创建一个基本的控制器，我们使用类和**装饰器**。装饰器将类与必需的元数据关联起来，并使 Nest 能够创建路由映射（将请求绑定到相应的控制器）。

> info **提示** 为了快速创建一个内置了[验证](https://docs.nestjs.com/techniques/validation)的 CRUD 控制器，你可以使用 CLI 的 [CRUD 生成器](https://docs.nestjs.com/recipes/crud-generator#crud-generator)：`nest g resource [name]`。

#### 路由

在下面的例子中，我们将使用 `@Controller()` 装饰器，这是**必需**的用于定义一个基本控制器。我们将指定一个可选的路径前缀 `cats`。在 `@Controller()` 装饰器中使用路径前缀可以让我们轻松地对一组相关的路由进行分组，并最大限度地减少重复代码。例如，我们可以选择将一组管理与猫实体交互的路由分组在 `/cats` 路由下。在这种情况下，我们可以在 `@Controller()` 装饰器中指定路径前缀 `cats`，这样就不必为文件中的每个路由重复该路径部分。

```typescript
@@filename(cats.controller)
import { Controller, Get } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Get()
  findAll(): string {
    return 'This action returns all cats';
  }
}
@@switch
import { Controller, Get } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Get()
  findAll() {
    return 'This action returns all cats';
  }
}
```

> info **提示** 要使用 CLI 创建控制器，只需执行 `$ nest g controller [name]` 命令。

`findAll()` 方法之前的 `@Get()` HTTP 请求方法装饰器告诉 Nest 为 HTTP 请求的特定端点创建一个处理程序。端点对应于 HTTP 请求方法（本例中为 GET）和路由路径。路由路径是什么？处理程序的路由路径由为控制器声明的（可选）前缀和方法的装饰器中指定的任何路径连接而成。由于我们为每个路由声明了一个前缀（`cats`），并且没有在装饰器中添加任何路径信息，因此 Nest 会将 `GET /cats` 请求映射到此处理程序。如前所述，路径包括可选的控制器路径前缀**和**请求方法装饰器中声明的任何路径字符串。例如，路径前缀 `cats` 与装饰器 `@Get('breed')` 组合将产生类似于 `GET /cats/breed` 的请求的路由映射。

在上面的例子中，当向此端点发出 GET 请求时，Nest 将请求路由到我们用户定义的 `findAll()` 方法。请注意，我们在这里选择的方法名称是任意的。显然，我们必须声明一个方法来绑定路由，但 Nest 不会对所选择的方法名称附加任何意义。

此方法将返回 200 状态码和相关的响应，在本例中只是一个字符串。为什么会这样？为了解释，我们首先介绍 Nest 使用两种**不同的**选项来操作响应的概念：

<table>
  <tr>
    <td>标准（推荐）</td>
    <td>
      使用这种内置方法，当请求处理程序返回 JavaScript 对象或数组时，它会<strong>自动</strong>序列化为 JSON。当它返回 JavaScript 原始类型（例如，<code>string</code>、<code>number</code>、<code>boolean</code>）时，Nest 将只发送该值而不尝试序列化它。这使得响应处理变得简单：只需返回值，Nest 会处理其余的事情。
      <br />
      <br /> 此外，响应的<strong>状态码</strong>默认始终为 200，除了使用 201 的 POST 请求。我们可以通过在处理程序级别添加 <code>@HttpCode(...)</code> 装饰器来轻松更改此行为（请参阅 <a href='controllers#status-code'>状态码</a>）。
    </td>
  </tr>
  <tr>
    <td>库特定</td>
    <td>
      我们可以使用库特定的（例如，Express）<a href="https://expressjs.com/en/api.html#res" rel="nofollow" target="_blank">响应对象</a>，可以通过在处理程序方法签名中使用 <code>@Res()</code> 装饰器注入（例如，<code>findAll(@Res() response)</code>）。 使用这种方法，你可以使用该对象暴露的本地响应处理方法。 例如，使用 Express，你可以使用类似 <code>response.status(200).send()</code> 的代码构建响应。
    </td>
  </tr>
</table>

> warning **警告** Nest 会检测处理程序是否使用 `@Res()` 或 `@Next()`，表明你选择了库特定的选项。如果同时使用两种方法，则**自动禁用**此单个路由的标准方法，并且将不再按预期工作。要同时使用两种方法（例如，通过注入响应对象仅用于设置 cookie/标头，但将其余部分留给框架），你必须在 `@Res({{ '{' }} passthrough: true {{ '}' }})` 装饰器中将 `passthrough` 选项设置为 `true`。

<app-banner-devtools></app-banner-devtools>

#### 请求对象

处理程序通常需要访问客户端**请求**的详细信息。Nest 提供对底层平台（默认为 Express）的[请求对象](https://expressjs.com/en/api.html#req)的访问。我们可以通过在处理程序的签名中添加 `@Req()` 装饰器来指示 Nest 注入请求对象，从而访问请求对象。

```typescript
@@filename(cats.controller)
import { Controller, Get, Req } from '@nestjs/common';
import { Request } from 'express';

@Controller('cats')
export class CatsController {
  @Get()
  findAll(@Req() request: Request): string {
    return 'This action returns all cats';
  }
}
@@switch
import { Controller, Bind, Get, Req } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Get()
  @Bind(Req())
  findAll(request) {
    return 'This action returns all cats';
  }
}
```

> info **提示** 为了利用 `express` 的类型（如上面 `request: Request` 参数示例所示），请安装 `@types/express` 包。

请求对象表示 HTTP 请求，并具有请求查询字符串、参数、HTTP 标头和正文的属性（在此处阅读更多内容[此处](https://expressjs.com/en/api.html#req)）。在大多数情况下，不需要手动获取这些属性。我们可以改用专用的装饰器，例如开箱即用的 `@Body()` 或 `@Query()`。下面是提供的装饰器及其代表的普通平台特定对象的列表。

<table>
  <tbody>
    <tr>
      <td><code>@Request(), @Req()</code></td>
      <td><code>req</code></td></tr>
    <tr>
      <td><code>@Response(), @Res()</code><span class="table-code-asterisk">*</span></td>
      <td><code>res</code></td>
    </tr>
    <tr>
      <td><code>@Next()</code></td>
      <td><code>next</code></td>
    </tr>
    <tr>
      <td><code>@Session()</code></td>
      <td><code>req.session</code></td>
    </tr>
    <tr>
      <td><code>@Param(key?: string)</code></td>
      <td><code>req.params</code> / <code>req.params[key]</code></td>
    </tr>
    <tr>
      <td><code>@Body(key?: string)</code></td>
      <td><code>req.body</code> / <code>req.body[key]</code></td>
    </tr>
    <tr>
      <td><code>@Query(key?: string)</code></td>
      <td><code>req.query</code> / <code>req.query[key]</code></td>
    </tr>
    <tr>
      <td><code>@Headers(name?: string)</code></td>
      <td><code>req.headers</code> / <code>req.headers[name]</code></td>
    </tr>
    <tr>
      <td><code>@Ip()</code></td>
      <td><code>req.ip</code></td>
    </tr>
    <tr>
      <td><code>@HostParam()</code></td>
      <td><code>req.hosts</code></td>
    </tr>
  </tbody>
</table>

<sup>\* </sup>为了与底层 HTTP 平台（例如 Express 和 Fastify）的类型兼容，Nest 提供了 `@Res()` 和 `@Response()` 装饰器。`@Res()` 只是 `@Response()` 的别名。两者都直接暴露底层本地平台 `response` 对象接口。使用它们时，你还应该导入底层库的类型（例如 `@types/express`）以充分利用它们。请注意，当你在方法处理程序中注入 `@Res()` 或 `@Response()` 时，你将 Nest 置于该处理程序的**库特定模式**，并且你负责管理响应。这样做时，你必须通过调用 `response` 对象（例如 `res.json(...)` 或 `res.send(...)`）发出某种响应，否则 HTTP 服务器将挂起。

> info **提示** 要了解如何创建自己的自定义装饰器，请访问[本章](/custom-decorators)。

#### 资源

之前，我们定义了一个端点来获取猫资源（**GET** 路由）。我们通常还希望提供一个创建新记录的端点。为此，让我们创建 **POST** 处理程序：

```typescript
@@filename(cats.controller)
import { Controller, Get, Post } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Post()
  create(): string {
    return 'This action adds a new cat';
  }

  @Get()
  findAll(): string {
    return 'This action returns all cats';
  }
}
@@switch
import { Controller, Get, Post } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Post()
  create() {
    return 'This action adds a new cat';
  }

  @Get()
  findAll() {
    return 'This action returns all cats';
  }
}
```

就是这么简单。Nest 为所有标准 HTTP 方法提供了装饰器：`@Get()`、`@Post()`、`@Put()`、`@Delete()`、`@Patch()`、`@Options()` 和 `@Head()`。此外，`@All()` 定义了一个处理所有这些方法的端点。

#### 路由通配符

NestJS 也支持基于模式的路由。例如，星号（`*`）可以用作通配符，以匹配路由路径末尾的任何字符组合。在以下示例中，`findAll()` 方法将为任何以 `abcd/` 开头的路由执行，无论后面跟着多少个字符。

```typescript
@Get('abcd/*')
findAll() {
  return 'This route uses a wildcard';
}
```

`'abcd/*'` 路由路径将匹配 `abcd/`、`abcd/123`、`abcd/abc` 等。连字符（`-`）和点（`.`）在基于字符串的路径中被按字面意义解释。

当星号用在**路由中间**时，Express 需要命名通配符（例如 `ab{{ '{' }}*splat&#125;cd`），而 Fastify 根本不支持它们。

#### 状态码

如前所述，响应**状态码**默认始终为 **200**，除了 POST 请求为 **201**。我们可以通过在处理程序级别添加 `@HttpCode(...)` 装饰器轻松更改此行为。

```typescript
@Post()
@HttpCode(204)
create() {
  return 'This action adds a new cat';
}
```

> info **提示** 从 `@nestjs/common` 包导入 `HttpCode`。

通常，你的状态码不是静态的，而是取决于各种因素。在这种情况下，你可以使用库特定的**响应**对象（通过 `@Res()` 注入）（或者，在发生错误的情况下，抛出异常）。

#### 标头

要指定自定义响应标头，你可以使用 `@Header()` 装饰器或库特定的响应对象（并直接调用 `res.header()`）。

```typescript
@Post()
@Header('Cache-Control', 'no-store')
create() {
  return 'This action adds a new cat';
}
```

> info **提示** 从 `@nestjs/common` 包导入 `Header`。

#### 重定向

要将响应重定向到特定的 URL，你可以使用 `@Redirect()` 装饰器或库特定的响应对象（并直接调用 `res.redirect()`）。

`@Redirect()` 接受两个参数，`url` 和 `statusCode`，两者都是可选的。如果省略，`statusCode` 的默认值为 `302`（`Found`）。

```typescript
@Get()
@Redirect('https://nestjs.com', 301)
```

> info **提示** 有时你可能希望动态确定 HTTP 状态码或重定向 URL。通过返回一个遵循 `HttpRedirectResponse` 接口（来自 `@nestjs/common`）的对象来完成此操作。

返回的值将覆盖传递给 `@Redirect()` 装饰器的任何参数。例如：

```typescript
@Get('docs')
@Redirect('https://docs.nestjs.com', 302)
getDocs(@Query('version') version) {
  if (version && version === '5') {
    return { url: 'https://docs.nestjs.com/v5/' };
  }
}
```

#### 路由参数

当你需要接受**动态数据**作为请求的一部分时（例如，`GET /cats/1` 获取 id 为 `1` 的猫），具有静态路径的路由将不起作用。为了定义带参数的路由，我们可以在路由的路径中添加路由参数**标记**，以捕获该位置在请求 URL 中的动态值。下面 `@Get()` 装饰器示例中的路由参数标记演示了这种用法。以这种方式声明的路由参数可以使用 `@Param()` 装饰器进行访问，该装饰器应添加到方法签名中。

> info **提示** 带参数的路由应在任何静态路径之后声明。这可以防止参数化路径拦截发往静态路径的流量。

```typescript
@@filename()
@Get(':id')
findOne(@Param() params: any): string {
  console.log(params.id);
  return `This action returns a #${params.id} cat`;
}
@@switch
@Get(':id')
@Bind(Param())
findOne(params) {
  console.log(params.id);
  return `This action returns a #${params.id} cat`;
}
```

`@Param()` 用于装饰方法参数（上例中的 `params`），并使**路由**参数作为该装饰方法参数的属性在方法体内可用。如上面的代码所示，我们可以通过引用 `params.id` 来访问 `id` 参数。你还可以将特定的参数标记传递给装饰器，然后在方法体中直接按名称引用路由参数。

> info **提示** 从 `@nestjs/common` 包导入 `Param`。

```typescript
@@filename()
@Get(':id')
findOne(@Param('id') id: string): string {
  return `This action returns a #${id} cat`;
}
@@switch
@Get(':id')
@Bind(Param('id'))
findOne(id) {
  return `This action returns a #${id} cat`;
}
```

#### 子域路由

`@Controller` 装饰器可以接受一个 `host` 选项，以要求传入请求的 HTTP 主机与某个特定值匹配。

```typescript
@Controller({ host: 'admin.example.com' })
export class AdminController {
  @Get()
  index(): string {
    return 'Admin page';
  }
}
```

> warning **警告** 由于 **Fastify** 不支持嵌套路由器，因此如果使用子域路由，建议改用默认的 Express 适配器。

与路由 `path` 类似，`hosts` 选项可以使用标记来捕获主机名中该位置的动态值。下面 `@Controller()` 装饰器示例中的主机参数标记演示了这种用法。以这种方式声明的主机参数可以使用 `@HostParam()` 装饰器进行访问，该装饰器应添加到方法签名中。

```typescript
@Controller({ host: ':account.example.com' })
export class AccountController {
  @Get()
  getInfo(@HostParam('account') account: string) {
    return account;
  }
}
```

#### 作用域

对于来自不同编程语言背景的人来说，可能会意外地了解到在 Nest 中，几乎所有内容都在传入请求之间共享。我们有一个到数据库的连接池，具有全局状态的单例服务等。请记住，Node.js 不遵循请求/响应多线程无状态模型，其中每个请求由单独的线程处理。因此，使用单例实例对我们的应用程序完全**安全**。

然而，在某些边缘情况下，基于请求的控制器生命周期可能是期望的行为，例如 GraphQL 应用程序中的每个请求缓存、请求跟踪或多租户。了解如何控制作用域[此处](/fundamentals/injection-scopes)。

#### 异步性

我们热爱现代 JavaScript，并且我们知道数据提取主要是**异步**的。这就是为什么 Nest 支持并与 `async` 函数良好配合。

> info **提示** 在此处了解更多关于 `async / await` 功能[此处](https://kamilmysliwiec.com/typescript-2-1-introduction-async-await)

每个异步函数都必须返回一个 `Promise`。这意味着你可以返回一个延迟值，Nest 将能够自行解析它。让我们看一个例子：

```typescript
@@filename(cats.controller)
@Get()
async findAll(): Promise<any[]> {
  return [];
}
@@switch
@Get()
async findAll() {
  return [];
}
```

上面的代码完全有效。此外，Nest 路由处理程序更强大，能够返回 RxJS [可观察流](https://rxjs-dev.firebaseapp.com/guide/observable)。Nest 将自动订阅底层的源并获取最后发出的值（一旦流完成）。

```typescript
@@filename(cats.controller)
@Get()
findAll(): Observable<any[]> {
  return of([]);
}
@@switch
@Get()
findAll() {
  return of([]);
}
```

上述两种方法都有效，你可以使用适合你要求的任何一种。

#### 请求负载

我们之前的 POST 路由处理程序示例没有接受任何客户端参数。让我们通过在此处添加 `@Body()` 装饰器来解决此问题。

但首先（如果你使用 TypeScript），我们需要确定 **DTO**（数据传输对象）模式。DTO 是一个定义数据如何通过网络发送的对象。我们可以使用 **TypeScript** 接口或简单的类来确定 DTO 模式。有趣的是，我们在这里推荐使用**类**。为什么？类是 JavaScript ES6 标准的一部分，因此它们在编译后的 JavaScript 中作为真实实体保留。另一方面，由于 TypeScript 接口在转译过程中被移除，Nest 在运行时无法引用它们。这很重要，因为诸如 **Pipes** 之类的功能在运行时可以访问变量的元类型时提供了额外的可能性。

让我们创建 `CreateCatDto` 类：

```typescript
@@filename(create-cat.dto)
export class CreateCatDto {
  name: string;
  age: number;
  breed: string;
}
```

它只有三个基本属性。此后我们可以在 `CatsController` 内部使用新创建的 DTO：

```typescript
@@filename(cats.controller)
@Post()
async create(@Body() createCatDto: CreateCatDto) {
  return 'This action adds a new cat';
}
@@switch
@Post()
@Bind(Body())
async create(createCatDto) {
  return 'This action adds a new cat';
}
```

> info **提示** 我们的 `ValidationPipe` 可以过滤掉不应被方法处理程序接收的属性。在这种情况下，我们可以将可接受的属性列入白名单，任何未包含在白名单中的属性将自动从结果对象中剥离。在 `CreateCatDto` 示例中，我们的白名单是 `name`、`age` 和 `breed` 属性。了解更多[此处](https://docs.nestjs.com/techniques/validation#stripping-properties)。

#### 处理错误

有一个关于处理错误（即处理异常）的单独章节[此处](/exception-filters)。

#### 完整资源示例

下面是一个示例，它利用了几个可用的装饰器来创建一个基本控制器。该控制器公开了一些方法来访问和操作内部数据。

```typescript
@@filename(cats.controller)
import { Controller, Get, Query, Post, Body, Put, Param, Delete } from '@nestjs/common';
import { CreateCatDto, UpdateCatDto, ListAllEntities } from './dto';

@Controller('cats')
export class CatsController {
  @Post()
  create(@Body() createCatDto: CreateCatDto) {
    return 'This action adds a new cat';
  }

  @Get()
  findAll(@Query() query: ListAllEntities) {
    return `This action returns all cats (limit: ${query.limit} items)`;
  }

  @Get(':id')
  findOne(@Param('id') id: string) {
    return `This action returns a #${id} cat`;
  }

  @Put(':id')
  update(@Param('id') id: string, @Body() updateCatDto: UpdateCatDto) {
    return `This action updates a #${id} cat`;
  }

  @Delete(':id')
  remove(@Param('id') id: string) {
    return `This action removes a #${id} cat`;
  }
}
@@switch
import { Controller, Get, Query, Post, Body, Put, Param, Delete, Bind } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Post()
  @Bind(Body())
  create(createCatDto) {
    return 'This action adds a new cat';
  }

  @Get()
  @Bind(Query())
  findAll(query) {
    console.log(query);
    return `This action returns all cats (limit: ${query.limit} items)`;
  }

  @Get(':id')
  @Bind(Param('id'))
  findOne(id) {
    return `This action returns a #${id} cat';
  }

  @Put(':id')
  @Bind(Param('id'), Body())
  update(id, updateCatDto) {
    return `This action updates a #${id} cat';
  }

  @Delete(':id')
  @Bind(Param('id'))
  remove(id) {
    return `This action removes a #${id} cat';
  }
}
```

> info **提示** Nest CLI 提供了一个生成器（原理图），可以自动生成**所有样板代码**，帮助我们避免所有这些工作，并使开发人员体验更加简单。阅读有关此功能的更多信息[此处](/recipes/crud-generator)。

#### 启动和运行

在完全定义了上述控制器之后，Nest 仍然不知道 `CatsController` 存在，因此不会创建此类的实例。

控制器始终属于一个模块，这就是为什么我们在 `@Module()` 装饰器中的 `controllers` 数组中包含它们的原因。由于我们尚未定义除根 `AppModule` 之外的任何其他模块，因此我们将使用它来引入 `CatsController`：

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { CatsController } from './cats/cats.controller';

@Module({
  controllers: [CatsController],
})
export class AppModule {}
```

我们使用 `@Module()` 装饰器将元数据附加到模块类，Nest 现在可以轻松反映必须挂载哪些控制器。

#### 库特定方法

到目前为止，我们已经讨论了 Nest 操作响应的标准方式。操作响应的第二种方法是使用库特定的[响应对象](https://expressjs.com/en/api.html#res)。为了注入特定的响应对象，我们需要使用 `@Res()` 装饰器。为了展示差异，让我们将 `CatsController` 重写为以下内容：

```typescript
@@filename()
import { Controller, Get, Post, Res, HttpStatus } from '@nestjs/common';
import { Response } from 'express';

@Controller('cats')
export class CatsController {
  @Post()
  create(@Res() res: Response) {
    res.status(HttpStatus.CREATED).send();
  }

  @Get()
  findAll(@Res() res: Response) {
     res.status(HttpStatus.OK).json([]);
  }
}
@@switch
import { Controller, Get, Post, Bind, Res, Body, HttpStatus } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Post()
  @Bind(Res(), Body())
  create(res, createCatDto) {
    res.status(HttpStatus.CREATED).send();
  }

  @Get()
  @Bind(Res())
  findAll(res) {
     res.status(HttpStatus.OK).json([]);
  }
}
```

尽管这种方法有效，并且实际上通过提供对响应对象的完全控制（标头操作、库特定功能等）在某些方面允许更大的灵活性，但应谨慎使用。一般来说，这种方法不太清晰，并且确实有一些缺点。主要缺点是代码变得依赖于平台（因为底层库在响应对象上可能具有不同的 API），并且更难以测试（你必须模拟响应对象等）。

此外，在上面的示例中，你失去了依赖于 Nest 标准响应处理的 Nest 功能的兼容性，例如拦截器和 `@HttpCode()` / `@Header()` 装饰器。要解决此问题，你可以将 `passthrough` 选项设置为 `true`，如下所示：

```typescript
@@filename()
@Get()
findAll(@Res({ passthrough: true }) res: Response) {
  res.status(HttpStatus.OK);
  return [];
}
@@switch
@Get()
@Bind(Res({ passthrough: true }))
findAll(res) {
  res.status(HttpStatus.OK);
  return [];
}
```

现在你可以与本地响应对象交互（例如，根据某些条件设置 cookie 或标头），但将其余部分留给框架。