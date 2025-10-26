### 其他功能

本页面列出了所有其他可用的功能，你可能会发现它们很有用。

#### 全局前缀

要忽略通过 `setGlobalPrefix()` 设置的全局前缀，可以使用 `ignoreGlobalPrefix`：

```typescript
const document = SwaggerModule.createDocument(app, options, {
  ignoreGlobalPrefix: true,
});
```

#### 全局参数

你可以使用 `DocumentBuilder` 为所有路由添加参数定义：

```typescript
const options = new DocumentBuilder().addGlobalParameters({
  name: 'tenantId',
  in: 'header',
});
```

#### 多规格支持

`SwaggerModule` 提供了支持多规格的方式。换句话说，你可以在不同的端点上提供不同的文档和不同的 UI。

要支持多规格，你的应用程序必须采用模块化的方式编写。`createDocument()` 方法接受第三个参数 `extraOptions`，它是一个具有 `include` 属性的对象。`include` 属性接受一个模块数组作为值。

你可以按如下方式设置多规格支持：

```typescript
import { NestFactory } from '@nestjs/core';
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';
import { AppModule } from './app.module';
import { CatsModule } from './cats/cats.module';
import { DogsModule } from './dogs/dogs.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  /**
   * createDocument(application, configurationOptions, extraOptions);
   *
   * createDocument 方法接受一个可选的第三个参数 "extraOptions"，
   * 它是一个具有 "include" 属性的对象，你可以在其中传递一个模块数组，
   * 这些模块将被包含在该 Swagger 规范中。
   * 例如：CatsModule 和 DogsModule 将有两个独立的 Swagger 规范，
   * 它们将在两个不同的端点上暴露两个不同的 SwaggerUI。
   */

  const options = new DocumentBuilder()
    .setTitle('Cats example')
    .setDescription('The cats API description')
    .setVersion('1.0')
    .addTag('cats')
    .build();

  const catDocumentFactory = () =>
    SwaggerModule.createDocument(app, options, {
      include: [CatsModule],
    });
  SwaggerModule.setup('api/cats', app, catDocumentFactory);

  const secondOptions = new DocumentBuilder()
    .setTitle('Dogs example')
    .setDescription('The dogs API description')
    .setVersion('1.0')
    .addTag('dogs')
    .build();

  const dogDocumentFactory = () =>
    SwaggerModule.createDocument(app, secondOptions, {
      include: [DogsModule],
    });
  SwaggerModule.setup('api/dogs', app, dogDocumentFactory);

  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

现在，你可以使用以下命令启动服务器：

```bash
$ npm run start
```

访问 `http://localhost:3000/api/cats` 查看猫咪的 Swagger UI：

<figure><img src="/assets/swagger-cats.png" /></figure>

而 `http://localhost:3000/api/dogs` 将暴露狗狗的 Swagger UI：

<figure><img src="/assets/swagger-dogs.png" /></figure>

#### 资源管理器栏中的下拉菜单

要在资源管理器栏的下拉菜单中启用多规格支持，你需要设置 `explorer: true` 并在 `SwaggerCustomOptions` 中配置 `swaggerOptions.urls`。

> 提示 **注意** 确保 `swaggerOptions.urls` 指向你的 Swagger 文档的 JSON 格式！要指定 JSON 文档，请在 `SwaggerCustomOptions` 中使用 `jsonDocumentUrl`。有关更多设置选项，请查看[此处](/openapi/introduction#setup-options)。

以下是如何在资源管理器栏中设置下拉菜单以支持多规格：

```typescript
import { NestFactory } from '@nestjs/core';
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';
import { AppModule } from './app.module';
import { CatsModule } from './cats/cats.module';
import { DogsModule } from './dogs/dogs.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // 主 API 选项
  const options = new DocumentBuilder()
    .setTitle('Multiple Specifications Example')
    .setDescription('Description for multiple specifications')
    .setVersion('1.0')
    .build();

  // 创建主 API 文档
  const document = SwaggerModule.createDocument(app, options);

  // 设置主 API Swagger UI 并支持下拉菜单
  SwaggerModule.setup('api', app, document, {
    explorer: true,
    swaggerOptions: {
      urls: [
        {
          name: '1. API',
          url: 'api/swagger.json',
        },
        {
          name: '2. Cats API',
          url: 'api/cats/swagger.json',
        },
        {
          name: '3. Dogs API',
          url: 'api/dogs/swagger.json',
        },
      ],
    },
    jsonDocumentUrl: '/api/swagger.json',
  });

  // 猫咪 API 选项
  const catOptions = new DocumentBuilder()
    .setTitle('Cats Example')
    .setDescription('Description for the Cats API')
    .setVersion('1.0')
    .addTag('cats')
    .build();

  // 创建猫咪 API 文档
  const catDocument = SwaggerModule.createDocument(app, catOptions, {
    include: [CatsModule],
  });

  // 设置猫咪 API Swagger UI
  SwaggerModule.setup('api/cats', app, catDocument, {
    jsonDocumentUrl: '/api/cats/swagger.json',
  });

  // 狗狗 API 选项
  const dogOptions = new DocumentBuilder()
    .setTitle('Dogs Example')
    .setDescription('Description for the Dogs API')
    .setVersion('1.0')
    .addTag('dogs')
    .build();

  // 创建狗狗 API 文档
  const dogDocument = SwaggerModule.createDocument(app, dogOptions, {
    include: [DogsModule],
  });

  // 设置狗狗 API Swagger UI
  SwaggerModule.setup('api/dogs', app, dogDocument, {
    jsonDocumentUrl: '/api/dogs/swagger.json',
  });

  await app.listen(3000);
}

bootstrap();
```

在这个例子中，我们设置了一个主 API 以及分别针对猫咪和狗狗的独立规范，每个都可以通过资源管理器栏中的下拉菜单访问。