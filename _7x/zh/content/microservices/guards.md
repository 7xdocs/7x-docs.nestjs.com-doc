### 守卫

微服务守卫与[常规HTTP应用程序守卫](/guards)之间没有根本区别。
唯一的区别是，你应该使用 `RpcException` 而不是抛出 `HttpException`。

> info **提示** `RpcException` 类来自 `@nestjs/microservices` 包。

#### 绑定守卫

下面的示例使用了方法作用域的守卫。与基于HTTP的应用程序一样，你也可以使用控制器作用域的守卫（即，在控制器类前添加 `@UseGuards()` 装饰器）。

```typescript
@@filename()
@UseGuards(AuthGuard)
@MessagePattern({ cmd: 'sum' })
accumulate(data: number[]): number {
  return (data || []).reduce((a, b) => a + b);
}
@@switch
@UseGuards(AuthGuard)
@MessagePattern({ cmd: 'sum' })
accumulate(data) {
  return (data || []).reduce((a, b) => a + b);
}
```