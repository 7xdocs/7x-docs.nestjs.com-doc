### 拦截器

[常规拦截器](/interceptors)和微服务拦截器之间没有区别。下面的示例使用了一个手动实例化的方法级拦截器。就像基于HTTP的应用程序一样，你也可以使用控制器级拦截器（即，在控制器类前添加`@UseInterceptors()`装饰器）。

```typescript
@@filename()
@UseInterceptors(new TransformInterceptor())
@MessagePattern({ cmd: 'sum' })
accumulate(data: number[]): number {
  return (data || []).reduce((a, b) => a + b);
}
@@switch
@UseInterceptors(new TransformInterceptor())
@MessagePattern({ cmd: 'sum' })
accumulate(data) {
  return (data || []).reduce((a, b) => a + b);
}
```