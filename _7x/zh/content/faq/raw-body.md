### 原始请求体

访问原始请求体最常见的用例之一是执行Webhook签名验证。通常，为了进行Webhook签名验证，需要未序列化的请求体来计算HMAC哈希。

> warning **警告** 此功能仅在启用内置的全局体解析器中间件时可用，即创建应用时不得传递`bodyParser: false`。

#### 在Express中使用

首先在创建Nest Express应用时启用该选项：

```typescript
import { NestFactory } from '@nestjs/core';
import type { NestExpressApplication } from '@nestjs/platform-express';
import { AppModule } from './app.module';

// 在 "bootstrap" 函数中
const app = await NestFactory.create<NestExpressApplication>(AppModule, {
  rawBody: true,
});
await app.listen(process.env.PORT ?? 3000);
```

要在控制器中访问原始请求体，Nest提供了一个便捷的接口`RawBodyRequest`，用于在请求上暴露`rawBody`字段：使用`RawBodyRequest`类型：

```typescript
import { Controller, Post, RawBodyRequest, Req } from '@nestjs/common';
import { Request } from 'express';

@Controller('cats')
class CatsController {
  @Post()
  create(@Req() req: RawBodyRequest<Request>) {
    const raw = req.rawBody; // 返回一个 `Buffer`。
  }
}
```

#### 注册不同的解析器

默认情况下，只注册了`json`和`urlencoded`解析器。如果想动态注册不同的解析器，需要显式进行操作。

例如，要注册`text`解析器，可以使用以下代码：

```typescript
app.useBodyParser('text');
```

> warning **警告** 确保为`NestFactory.create`调用提供了正确的应用类型。对于Express应用，正确的类型是`NestExpressApplication`。否则将找不到`.useBodyParser`方法。

#### 体解析器大小限制

如果你的应用需要解析比Express默认的`100kb`更大的请求体，请使用以下代码：

```typescript
app.useBodyParser('json', { limit: '10mb' });
```

`.useBodyParser`方法会尊重在应用选项中传递的`rawBody`选项。

#### 在Fastify中使用

首先在创建Nest Fastify应用时启用该选项：

```typescript
import { NestFactory } from '@nestjs/core';
import {
  FastifyAdapter,
  NestFastifyApplication,
} from '@nestjs/platform-fastify';
import { AppModule } from './app.module';

// 在 "bootstrap" 函数中
const app = await NestFactory.create<NestFastifyApplication>(
  AppModule,
  new FastifyAdapter(),
  {
    rawBody: true,
  },
);
await app.listen(process.env.PORT ?? 3000);
```

要在控制器中访问原始请求体，Nest提供了一个便捷的接口`RawBodyRequest`，用于在请求上暴露`rawBody`字段：使用`RawBodyRequest`类型：

```typescript
import { Controller, Post, RawBodyRequest, Req } from '@nestjs/common';
import { FastifyRequest } from 'fastify';

@Controller('cats')
class CatsController {
  @Post()
  create(@Req() req: RawBodyRequest<FastifyRequest>) {
    const raw = req.rawBody; // 返回一个 `Buffer`。
  }
}
```

#### 注册不同的解析器

默认情况下，只注册了`application/json`和`application/x-www-form-urlencoded`解析器。如果想动态注册不同的解析器，需要显式进行操作。

例如，要注册`text/plain`解析器，可以使用以下代码：

```typescript
app.useBodyParser('text/plain');
```

> warning **警告** 确保为`NestFactory.create`调用提供了正确的应用类型。对于Fastify应用，正确的类型是`NestFastifyApplication`。否则将找不到`.useBodyParser`方法。

#### 体解析器大小限制

如果你的应用需要解析比Fastify默认的1MiB更大的请求体，请使用以下代码：

```typescript
const bodyLimit = 10_485_760; // 10MiB
app.useBodyParser('application/json', { bodyLimit });
```

`.useBodyParser`方法会尊重在应用选项中传递的`rawBody`选项。