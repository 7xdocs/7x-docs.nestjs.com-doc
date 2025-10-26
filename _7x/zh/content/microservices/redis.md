### Redis

[Redis](https://redis.io/) 传输器实现了发布/订阅消息范式，并利用了Redis的[Pub/Sub](https://redis.io/topics/pubsub)功能。已发布的消息按频道分类，无需知道最终会有哪些订阅者（如果有的话）接收消息。每个微服务可以订阅任意数量的频道。此外，一次可以订阅多个频道。通过频道交换的消息是**即发即弃**的，这意味着如果一条消息被发布但没有感兴趣的订阅者，该消息会被移除且无法恢复。因此，不能保证消息或事件会被至少一个服务处理。一条消息可以被多个订阅者订阅（并接收）。

<figure><img class="illustrative-image" src="/assets/Redis_1.png" /></figure>

#### 安装

要开始构建基于Redis的微服务，首先安装所需的包：

```bash
$ npm i --save ioredis
```

#### 概述

要使用Redis传输器，请将以下选项对象传递给`createMicroservice()`方法：

```typescript
@@filename(main)
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.REDIS,
  options: {
    host: 'localhost',
    port: 6379,
  },
});
@@switch
const app = await NestFactory.createMicroservice(AppModule, {
  transport: Transport.REDIS,
  options: {
    host: 'localhost',
    port: 6379,
  },
});
```

> info **提示** `Transport`枚举从`@nestjs/microservices`包导入。

#### 选项

`options`属性特定于所选的传输器。**Redis**传输器提供以下属性：

<table>
  <tr>
    <td><code>host</code></td>
    <td>连接URL</td>
  </tr>
  <tr>
    <td><code>port</code></td>
    <td>连接端口</td>
  </tr>
  <tr>
    <td><code>retryAttempts</code></td>
    <td>消息重试次数（默认：<code>0</code>）</td>
  </tr>
  <tr>
    <td><code>retryDelay</code></td>
    <td>消息重试尝试之间的延迟（毫秒）（默认：<code>0</code>）</td>
  </tr>
   <tr>
    <td><code>wildcards</code></td>
    <td>启用Redis通配符订阅，指示传输器在底层使用<code>psubscribe</code>/<code>pmessage</code>（默认：<code>false</code>）</td>
  </tr>
</table>

官方[ioredis](https://redis.github.io/ioredis/index.html#RedisOptions)客户端支持的所有属性也受此传输器支持。

#### 客户端

与其他微服务传输器一样，创建Redis`ClientProxy`实例有[多种方式](https://docs.nestjs.com/microservices/basics#client)。

创建实例的一种方法是使用`ClientsModule`。要通过`ClientsModule`创建客户端实例，请导入它并使用`register()`方法传递一个选项对象（该对象包含与上述`createMicroservice()`方法中所示相同的属性），以及一个用作注入令牌的`name`属性。有关`ClientsModule`的更多信息，请[查看此处](https://docs.nestjs.com/microservices/basics#client)。

```typescript
@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'MATH_SERVICE',
        transport: Transport.REDIS,
        options: {
          host: 'localhost',
          port: 6379,
        }
      },
    ]),
  ]
  ...
})
```

也可以使用其他方式创建客户端（`ClientProxyFactory`或`@Client()`）。你可以[在此处](https://docs.nestjs.com/microservices/basics#client)了解相关内容。

#### 上下文

在更复杂的场景中，你可能需要访问有关传入请求的额外信息。使用Redis传输器时，可以访问`RedisContext`对象。

```typescript
@@filename()
@MessagePattern('notifications')
getNotifications(@Payload() data: number[], @Ctx() context: RedisContext) {
  console.log(`Channel: ${context.getChannel()}`);
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('notifications')
getNotifications(data, context) {
  console.log(`Channel: ${context.getChannel()}`);
}
```

> info **提示** `@Payload()`、`@Ctx()`和`RedisContext`从`@nestjs/microservices`包导入。

#### 通配符

要启用通配符支持，请将`wildcards`选项设置为`true`。这会指示传输器在底层使用`psubscribe`和`pmessage`。

```typescript
const app = await NestFactory.createMicroservice(AppModule, {
  transport: Transport.REDIS,
  options: {
    // 其他选项
    wildcards: true,
  },
});
```

创建客户端实例时，确保也传递`wildcards`选项。

启用此选项后，可以在消息和事件模式中使用通配符。例如，要订阅所有以`notifications`开头的频道，可以使用以下模式：

```typescript
@EventPattern('notifications.*')
```

#### 实例状态更新

要获取有关连接和底层驱动实例状态的实时更新，可以订阅`status`流。此流提供特定于所选驱动的状态更新。对于Redis驱动，`status`流会发出`connected`、`disconnected`和`reconnecting`事件。

```typescript
this.client.status.subscribe((status: RedisStatus) => {
  console.log(status);
});
```

> info **提示** RedisStatus类型从`@nestjs/microservices`包导入。

同样，可以订阅服务器的`status`流以接收有关服务器状态的通知。

```typescript
const server = app.connectMicroservice<MicroserviceOptions>(...);
server.status.subscribe((status: RedisStatus) => {
  console.log(status);
});
```

#### 监听Redis事件

在某些情况下，你可能希望监听微服务发出的内部事件。例如，你可以监听`error`事件，以便在发生错误时触发额外操作。为此，请使用`on()`方法，如下所示：

```typescript
this.client.on('error', (err) => {
  console.error(err);
});
```

同样，可以监听服务器的内部事件：

```typescript
server.on<RedisEvents>('error', (err) => {
  console.error(err);
});
```

> info **提示** RedisEvents类型从`@nestjs/microservices`包导入。

#### 访问底层驱动

对于更高级的用例，你可能需要访问底层驱动实例。这在诸如手动关闭连接或使用驱动特定方法等场景中可能很有用。但是，请记住，在大多数情况下，你**不需要**直接访问驱动。

要访问底层驱动，可以使用`unwrap()`方法，该方法返回底层驱动实例。泛型类型参数应指定你期望的驱动实例类型。

```typescript
const [pub, sub] =
  this.client.unwrap<[import('ioredis').Redis, import('ioredis').Redis]>();
```

同样，可以访问服务器的底层驱动实例：

```typescript
const [pub, sub] =
  server.unwrap<[import('ioredis').Redis, import('ioredis').Redis]>();
```

请注意，与其他传输器不同，Redis传输器返回两个`ioredis`实例组成的元组：第一个用于发布消息，第二个用于订阅消息。