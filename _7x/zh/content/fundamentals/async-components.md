### 异步提供者

有时候，应用程序的启动需要延迟到某些**异步任务**完成之后。例如，您可能希望在建立与数据库的连接之前不开始接受请求。您可以使用异步提供者来实现这一点。

实现这一点的语法是在 `useFactory` 语法中使用 `async/await`。工厂函数返回一个 `Promise`，并且工厂函数可以 `await` 异步任务。在实例化任何依赖于（注入）此类提供者的类之前，Nest 将等待 Promise 的解析。

```typescript
{
  provide: 'ASYNC_CONNECTION',
  useFactory: async () => {
    const connection = await createConnection(options);
    return connection;
  },
}
```

> info **提示** 在此处了解更多关于[自定义提供者](/fundamentals/custom-providers)的语法。

#### 注入

异步提供者像任何其他提供者一样，通过它们的令牌注入到其他组件中。在上面的示例中，您将使用 `@Inject('ASYNC_CONNECTION')` 构造。

#### 示例

[TypeORM 集成章节](/recipes/sql-typeorm)提供了一个更详尽的异步提供者示例。