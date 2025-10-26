### 压缩

压缩可以显著减小响应体的大小，从而提高 Web 应用程序的速度。

对于生产环境中的**高流量**网站，强烈建议将压缩任务从应用服务器卸载——通常是在反向代理（例如 Nginx）中处理。在这种情况下，你不应使用压缩中间件。

#### 在 Express 中使用（默认）

使用 [compression](https://github.com/expressjs/compression) 中间件包来启用 gzip 压缩。

首先安装所需的包：

```bash
$ npm i --save compression
```

安装完成后，将压缩中间件应用为全局中间件。

```typescript
import * as compression from 'compression';
// somewhere in your initialization file
app.use(compression());
```

#### 在 Fastify 中使用

如果使用 `FastifyAdapter`，你需要使用 [fastify-compress](https://github.com/fastify/fastify-compress)：

```bash
$ npm i --save @fastify/compress
```

安装完成后，将 `@fastify/compress` 中间件应用为全局中间件。

```typescript
import compression from '@fastify/compress';
// somewhere in your initialization file
await app.register(compression);
```

默认情况下，当浏览器表明支持该编码时，`@fastify/compress` 将使用 Brotli 压缩（在 Node >= 11.7.0 上）。虽然 Brotli 在压缩率方面非常高效，但它也可能非常慢。默认情况下，Brotli 设置最大压缩质量为 11，但可以通过调整 `BROTLI_PARAM_QUALITY` 在 0（最小）到 11（最大）之间进行调整，以牺牲压缩质量来减少压缩时间。这需要微调以优化空间/时间性能。一个质量为 4 的示例：

```typescript
import { constants } from 'zlib';
// somewhere in your initialization file
await app.register(compression, { brotliOptions: { params: { [constants.BROTLI_PARAM_QUALITY]: 4 } } });
```

为了简化，你可能想告诉 `fastify-compress` 仅使用 deflate 和 gzip 来压缩响应；这样可能会导致响应体积变大，但它们的传递速度会快得多。

要指定编码，请向 `app.register` 提供第二个参数：

```typescript
await app.register(compression, { encodings: ['gzip', 'deflate'] });
```

以上代码告诉 `fastify-compress` 仅使用 gzip 和 deflate 编码，如果客户端同时支持两者，则优先使用 gzip。