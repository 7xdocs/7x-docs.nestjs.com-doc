### 管道

常规管道（[regular pipes](/pipes)）和微服务管道之间没有根本区别。唯一的区别是，你应该使用 `RpcException` 而不是抛出 `HttpException`。

> info **提示** `RpcException` 类来自 `@nestjs/microservices` 包。

#### 绑定管道

以下示例使用了一个手动实例化的方法作用域管道。与基于HTTP的应用程序一样，你也可以使用控制器作用域管道（即，在控制器类前添加 `@UsePipes()` 装饰器）。

```typescript
@@filename()
@UsePipes(new ValidationPipe({ exceptionFactory: (errors) => new RpcException(errors) }))
@MessagePattern({ cmd: 'sum' })
accumulate(data: number[]): number {
  return (data || []).reduce((a, b) => a + b);
}
@@switch
@UsePipes(new ValidationPipe({ exceptionFactory: (errors) => new RpcException(errors) }))
@MessagePattern({ cmd: 'sum' })
accumulate(data) {
  return (data || []).reduce((a, b) => a + b);
}
```