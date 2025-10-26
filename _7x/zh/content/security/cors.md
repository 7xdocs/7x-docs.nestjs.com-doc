### CORS

跨源资源共享（CORS）是一种允许从另一个域请求资源的机制。在底层，Nest 根据所使用的底层平台，使用了 Express 的 [cors](https://github.com/expressjs/cors) 包或 Fastify 的 [@fastify/cors](https://github.com/fastify/fastify-cors) 包。这些包提供了多种选项，您可以根据需求进行自定义。

#### 开始使用

要启用 CORS，请在 Nest 应用程序对象上调用 `enableCors()` 方法。

```typescript
const app = await NestFactory.create(AppModule);
app.enableCors();
await app.listen(process.env.PORT ?? 3000);
```

`enableCors()` 方法接受一个可选的配置对象参数。该对象的可用属性在官方 [CORS](https://github.com/expressjs/cors#configuration-options) 文档中有详细说明。另一种方式是传递一个[回调函数](https://github.com/expressjs/cors#configuring-cors-asynchronously)，该函数允许您根据请求动态地异步定义配置对象。

或者，可以通过 `create()` 方法的选项对象来启用 CORS。将 `cors` 属性设置为 `true` 即可使用默认设置启用 CORS。
或者，将 [CORS 配置对象](https://github.com/expressjs/cors#configuration-options) 或 [回调函数](https://github.com/expressjs/cors#configuring-cors-asynchronously) 作为 `cors` 属性的值传入，以自定义其行为。

```typescript
const app = await NestFactory.create(AppModule, { cors: true });
await app.listen(process.env.PORT ?? 3000);
```