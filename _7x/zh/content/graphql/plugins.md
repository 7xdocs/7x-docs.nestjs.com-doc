### 插件

插件使您能够通过在某些事件发生时执行自定义操作来扩展 Apollo Server 的核心功能。目前，这些事件对应于 GraphQL 请求生命周期的各个阶段，以及 Apollo Server 本身的启动（更多信息请参见[此处](https://www.apollographql.com/docs/apollo-server/integrations/plugins/)）。例如，一个基础的日志插件可以记录发送到 Apollo Server 的每个请求所关联的 GraphQL 查询字符串。

#### 自定义插件

要创建一个插件，需要声明一个使用从 `@nestjs/apollo` 包导出的 `@Plugin` 装饰器注解的类。此外，为了获得更好的代码自动完成功能，可以实现来自 `@apollo/server` 包的 `ApolloServerPlugin` 接口。

```typescript
import { ApolloServerPlugin, GraphQLRequestListener } from '@apollo/server';
import { Plugin } from '@nestjs/apollo';

@Plugin()
export class LoggingPlugin implements ApolloServerPlugin {
  async requestDidStart(): Promise<GraphQLRequestListener<any>> {
    console.log('Request started');
    return {
      async willSendResponse() {
        console.log('Will send response');
      },
    };
  }
}
```

完成这些后，我们可以将 `LoggingPlugin` 注册为一个提供者。

```typescript
@Module({
  providers: [LoggingPlugin],
})
export class CommonModule {}
```

Nest 将自动实例化该插件并将其应用到 Apollo Server。

#### 使用外部插件

Apollo 提供了一些现成的插件。要使用现有插件，只需导入它并将其添加到 `plugins` 数组中：

```typescript
GraphQLModule.forRoot({
  // ...
  plugins: [ApolloServerOperationRegistry({ /* options */})]
}),
```

> info **提示** `ApolloServerOperationRegistry` 插件是从 `@apollo/server-plugin-operation-registry` 包导出的。

#### 与 Mercurius 一起使用插件

一些现有的特定于 mercurius 的 Fastify 插件必须在插件树中的 mercurius 插件之后加载（更多信息请参见[此处](https://mercurius.dev/#/docs/plugins)）。

> warning **警告** [mercurius-upload](https://github.com/mercurius-js/mercurius-upload) 是一个例外，应该在主文件中注册。

为此，`MercuriusDriver` 暴露了一个可选的 `plugins` 配置选项。它代表一个对象数组，每个对象包含两个属性：`plugin` 及其 `options`。因此，注册[缓存插件](https://github.com/mercurius-js/cache) 将如下所示：

```typescript
GraphQLModule.forRoot({
  driver: MercuriusDriver,
  // ...
  plugins: [
    {
      plugin: cache,
      options: {
        ttl: 10,
        policy: {
          Query: {
            add: true
          }
        }
      },
    }
  ]
}),
```