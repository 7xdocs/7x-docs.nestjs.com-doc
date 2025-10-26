### 守卫

Web Sockets 守卫与[常规 HTTP 应用程序守卫](/guards)之间没有根本区别。唯一的区别是，你应该使用 `WsException` 而不是抛出 `HttpException`。

> info **提示** `WsException` 类是从 `@nestjs/websockets` 包中导出的。

#### 绑定守卫

下面的例子使用了一个方法作用域的守卫。就像基于 HTTP 的应用程序一样，你也可以使用网关作用域的守卫（例如，在网关类上使用 `@UseGuards()` 装饰器）。

```typescript
@@filename()
@UseGuards(AuthGuard)
@SubscribeMessage('events')
handleEvent(client: Client, data: unknown): WsResponse<unknown> {
  const event = 'events';
  return { event, data };
}
@@switch
@UseGuards(AuthGuard)
@SubscribeMessage('events')
handleEvent(client, data) {
  const event = 'events';
  return { event, data };
}
```