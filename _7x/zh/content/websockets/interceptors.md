### 拦截器

[常规拦截器](/interceptors)与 Web Socket 拦截器之间没有区别。以下示例使用了一个手动实例化的方法作用域拦截器。就像基于 HTTP 的应用一样，你也可以使用网关作用域的拦截器（例如，在网关类前添加 `@UseInterceptors()` 装饰器）。

```typescript
@@filename()
@UseInterceptors(new TransformInterceptor())
@SubscribeMessage('events')
handleEvent(client: Client, data: unknown): WsResponse<unknown> {
  const event = 'events';
  return { event, data };
}
@@switch
@UseInterceptors(new TransformInterceptor())
@SubscribeMessage('events')
handleEvent(client, data) {
  const event = 'events';
  return { event, data };
}
```