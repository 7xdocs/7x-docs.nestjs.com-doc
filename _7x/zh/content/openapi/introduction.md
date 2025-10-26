### 简介

[OpenAPI](https://swagger.io/specification/) 规范是一种与编程语言无关的定义格式，用于描述 RESTful API。Nest 提供了一个专用的[模块](https://github.com/nestjs/swagger)，允许通过装饰器来生成此类规范。

#### 安装

要开始使用，我们首先安装所需的依赖。

```bash
$ npm install --save @nestjs/swagger
```

#### 初始化

安装过程完成后，打开 `main.ts` 文件，使用 `SwaggerModule` 类初始化 Swagger：

```typescript
@@filename(main)
import { NestFactory } from '@nestjs/core';
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  const config = new DocumentBuilder()
    .setTitle('Cats example')
    .setDescription('The cats API description')
    .setVersion('1.0')
    .addTag('cats')
    .build();
  const documentFactory = () => SwaggerModule.createDocument(app, config);
  SwaggerModule.setup('api', app, documentFactory);

  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

> info **提示** `SwaggerModule#createDocument()` 工厂方法专门用于在您请求时生成 Swagger 文档。这种方法有助于节省一些初始化时间，生成的文档是一个符合 [OpenAPI 文档](https://swagger.io/specification/#openapi-document) 规范的可序列化对象。除了通过 HTTP 提供文档外，您还可以将其保存为 JSON 或 YAML 文件，并以各种方式使用。

`DocumentBuilder` 有助于构建符合 OpenAPI 规范的基础文档。它提供了多个方法，允许设置诸如标题、描述、版本等属性。为了创建完整的文档（包含所有已定义的 HTTP 路由），我们使用 `SwaggerModule` 类的 `createDocument()` 方法。此方法接受两个参数：一个应用程序实例和一个 Swagger 选项对象。或者，我们可以提供第三个参数，其类型应为 `SwaggerDocumentOptions`。更多内容请参阅 [文档选项部分](/openapi/introduction#document-options)。

创建文档后，我们可以调用 `setup()` 方法。它接受：

1. 挂载 Swagger UI 的路径
2. 应用程序实例
3. 上面实例化的文档对象
4. 可选的配置参数（更多信息请阅读 [此处](/openapi/introduction#setup-options)）

现在，您可以运行以下命令来启动 HTTP 服务器：

```bash
$ npm run start
```

应用程序运行时，打开浏览器并导航到 `http://localhost:3000/api`。您应该会看到 Swagger UI。

<figure><img src="/assets/swagger1.png" /></figure>

如您所见，`SwaggerModule` 自动反射了您的所有端点。

> info **提示** 要生成并下载 Swagger JSON 文件，请导航到 `http://localhost:3000/api-json`（假设您的 Swagger 文档在 `http://localhost:3000/api` 下可用）。
> 也可以仅使用 `@nestjs/swagger` 中的 setup 方法，在您选择的路由上公开它，如下所示：
>
> ```typescript
> SwaggerModule.setup('swagger', app, document, {
>   jsonDocumentUrl: 'swagger/json',
> });
> ```
>
> 这将在 `http://localhost:3000/swagger/json` 处公开它。

> warning **警告** 当使用 `fastify` 和 `helmet` 时，可能会与 [CSP](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP) 发生问题。为了解决这个冲突，请按如下方式配置 CSP：
>
> ```typescript
> app.register(helmet, {
>   contentSecurityPolicy: {
>     directives: {
>       defaultSrc: [`'self'`],
>       styleSrc: [`'self'`, `'unsafe-inline'`],
>       imgSrc: [`'self'`, 'data:', 'validator.swagger.io'],
>       scriptSrc: [`'self'`, `https: 'unsafe-inline'`],
>     },
>   },
> });
>
> // 如果您完全不打算使用 CSP，可以这样设置：
> app.register(helmet, {
>   contentSecurityPolicy: false,
> });
> ```

#### 文档选项

创建文档时，可以提供一些额外的选项来微调库的行为。这些选项的类型应为 `SwaggerDocumentOptions`，可以包含以下属性：

```TypeScript
export interface SwaggerDocumentOptions {
  /**
   * 要包含在规范中的模块列表
   */
  include?: Function[];

  /**
   * 应被检查并包含在规范中的额外的、额外的模型
   */
  extraModels?: Function[];

  /**
   * 如果为 `true`，swagger 将忽略通过 `setGlobalPrefix()` 方法设置的全局前缀
   */
  ignoreGlobalPrefix?: boolean;

  /**
   * 如果为 `true`，swagger 还将加载由 `include` 模块导入的模块中的路由
   */
  deepScanRoutes?: boolean;

  /**
   * 用于生成 `operationId` 的自定义 operationIdFactory，基于 `controllerKey`、`methodKey` 和版本。
   * @default () => controllerKey_methodKey_version
   */
  operationIdFactory?: OperationIdFactory;

  /**
   * 用于生成响应中 `links` 字段链接名称的自定义 linkNameFactory
   *
   * @see [链接对象](https://swagger.io/docs/specification/links/)
   *
   * @default () => `${controllerKey}_${methodKey}_from_${fieldKey}`
   */
  linkNameFactory?: (
    controllerKey: string,
    methodKey: string,
    fieldKey: string
  ) => string;

  /*
   * 基于控制器名称自动生成标签。
   * 如果为 `false`，则必须使用 `@ApiTags()` 装饰器来定义标签。
   * 否则，将使用不带 `Controller` 后缀的控制器名称。
   * @default true
   */
  autoTagControllers?: boolean;
}
```

例如，如果您希望确保库生成像 `createUser` 这样的操作名，而不是 `UserController_createUser`，您可以进行如下设置：

```TypeScript
const options: SwaggerDocumentOptions =  {
  operationIdFactory: (
    controllerKey: string,
    methodKey: string
  ) => methodKey
};
const documentFactory = () => SwaggerModule.createDocument(app, config, options);
```

#### 设置选项

您可以通过将满足 `SwaggerCustomOptions` 接口的选项对象作为 `SwaggerModule#setup` 方法的第四个参数来配置 Swagger UI。

```TypeScript
export interface SwaggerCustomOptions {
  /**
   * 如果为 `true`，Swagger 资源路径将加上通过 `setGlobalPrefix()` 设置的全局前缀。
   * 默认值：`false`。
   * @see https://docs.nestjs.com/faq/global-prefix
   */
  useGlobalPrefix?: boolean;

  /**
   * 如果为 `false`，将仅提供 API 定义（JSON 和 YAML）（在 `/{path}-json` 和 `/{path}-yaml` 上）。
   * 如果您已经在其他地方托管了 Swagger UI 并且只想提供 API 定义，这将特别有用。
   * 默认值：`true`。
   */
  swaggerUiEnabled?: boolean;

  /**
   * 指向要在 Swagger UI 中加载的 API 定义的 URL。
   */
  swaggerUrl?: string;

  /**
   * 要提供的 JSON API 定义的路径。
   * 默认值：`<path>-json`。
   */
  jsonDocumentUrl?: string;

  /**
   * 要提供的 YAML API 定义的路径。
   * 默认值：`<path>-yaml`。
   */
  yamlDocumentUrl?: string;

  /**
   * 在提供 OpenAPI 文档之前允许修改它的钩子。
   * 它在文档生成之后、作为 JSON 和 YAML 提供之前被调用。
   */
  patchDocumentOnRequest?: <TRequest = any, TResponse = any>(
    req: TRequest,
    res: TResponse,
    document: OpenAPIObject
  ) => OpenAPIObject;

  /**
   * 如果为 `true`，OpenAPI 定义的选择器将显示在 Swagger UI 界面中。
   * 默认值：`false`。
   */
  explorer?: boolean;

  /**
   * 额外的 Swagger UI 选项
   */
  swaggerOptions?: SwaggerUiOptions;

  /**
   * 要注入到 Swagger UI 页面中的自定义 CSS 样式。
   */
  customCss?: string;

  /**
   * 要在 Swagger UI 页面中加载的自定义 CSS 样式表的 URL。
   */
  customCssUrl?: string | string[];

  /**
   * 要在 Swagger UI 页面中加载的自定义 JavaScript 文件的 URL。
   */
  customJs?: string | string[];

  /**
   * 要在 Swagger UI 页面中加载的自定义 JavaScript 脚本。
   */
  customJsStr?: string | string[];

  /**
   * Swagger UI 页面的自定义网站图标。
   */
  customfavIcon?: string;

  /**
   * Swagger UI 页面的自定义标题。
   */
  customSiteTitle?: string;

  /**
   * 包含静态 Swagger UI 资源的文件系统路径（例如：./node_modules/swagger-ui-dist）。
   */
  customSwaggerUiPath?: string;

  /**
   * @deprecated 此属性无效。
   */
  validatorUrl?: string;

  /**
   * @deprecated 此属性无效。
   */
  url?: string;

  /**
   * @deprecated 此属性无效。
   */
  urls?: Record<'url' | 'name', string>[];

}
```

#### 示例

可用的工作示例在[这里](https://github.com/nestjs/nest/tree/master/sample/11-swagger)。