### RabbitMQ

[RabbitMQ](https://www.rabbitmq.com/) 是一个开源的轻量级消息代理，支持多种消息传递协议。它可以部署在分布式和联邦配置中，以满足高规模、高可用性需求。此外，它是应用最广泛的消息代理，在全球范围内的小型初创公司和大型企业中都有使用。

#### 安装

要开始构建基于RabbitMQ的微服务，首先安装所需的包：

```bash
$ npm i --save amqplib amqp-connection-manager
```

#### 概述

要使用RabbitMQ传输器，请将以下选项对象传递给`createMicroservice()`方法：

```typescript
@@filename(main)
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.RMQ,
  options: {
    urls: ['amqp://localhost:5672'],
    queue: 'cats_queue',
    queueOptions: {
      durable: false
    },
  },
});
@@switch
const app = await NestFactory.createMicroservice(AppModule, {
  transport: Transport.RMQ,
  options: {
    urls: ['amqp://localhost:5672'],
    queue: 'cats_queue',
    queueOptions: {
      durable: false
    },
  },
});
```

> info **提示** `Transport`枚举从`@nestjs/microservices`包导入。

#### 选项

`options`属性特定于所选的传输器。**RabbitMQ**传输器公开以下属性。

<table>
  <tr>
    <td><code>urls</code></td>
    <td>连接URLs</td>
  </tr>
  <tr>
    <td><code>queue</code></td>
    <td>服务器将监听的队列名称</td>
  </tr>
  <tr>
    <td><code>prefetchCount</code></td>
    <td>设置通道的预取计数</td>
  </tr>
  <tr>
    <td><code>isGlobalPrefetchCount</code></td>
    <td>启用按通道预取</td>
  </tr>
  <tr>
    <td><code>noAck</code></td>
    <td>如果为<code>false</code>，则启用手动确认模式</td>
  </tr>
  <tr>
    <td><code>consumerTag</code></td>
    <td>消费者标签标识符（更多信息见<a href="https://amqp-node.github.io/amqplib/channel_api.html#channel_consume" rel="nofollow" target="_blank">此处</a>）</td>
  </tr>
  <tr>
    <td><code>queueOptions</code></td>
    <td>附加队列选项（更多信息见<a href="https://amqp-node.github.io/amqplib/channel_api.html#channel_assertQueue" rel="nofollow" target="_blank">此处</a>）</td>
  </tr>
  <tr>
    <td><code>socketOptions</code></td>
    <td>附加套接字选项（更多信息见<a href="https://amqp-node.github.io/amqplib/channel_api.html#connect" rel="nofollow" target="_blank">此处</a>）</td>
  </tr>
  <tr>
    <td><code>headers</code></td>
    <td>随每条消息一起发送的头信息</td>
  </tr>
</table>

#### 客户端

与其他微服务传输器一样，创建RabbitMQ`ClientProxy`实例有<a href="https://docs.nestjs.com/microservices/basics#client">多种选项</a>。

创建实例的一种方法是使用`ClientsModule`。要使用`ClientsModule`创建客户端实例，请导入它并使用`register()`方法传递一个选项对象，该对象具有与上面`createMicroservice()`方法中所示相同的属性，以及一个用作注入令牌的`name`属性。在此处了解有关`ClientsModule`的更多信息<a href="https://docs.nestjs.com/microservices/basics#client">此处</a>。

```typescript
@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'MATH_SERVICE',
        transport: Transport.RMQ,
        options: {
          urls: ['amqp://localhost:5672'],
          queue: 'cats_queue',
          queueOptions: {
            durable: false
          },
        },
      },
    ]),
  ]
  ...
})
```

也可以使用其他创建客户端的选项（`ClientProxyFactory`或`@Client()`）。你可以在<a href="https://docs.nestjs.com/microservices/basics#client">此处</a>了解它们。

#### 上下文

在更复杂的场景中，你可能需要访问有关传入请求的附加信息。使用RabbitMQ传输器时，你可以访问`RmqContext`对象。

```typescript
@@filename()
@MessagePattern('notifications')
getNotifications(@Payload() data: number[], @Ctx() context: RmqContext) {
  console.log(`Pattern: ${context.getPattern()}`);
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('notifications')
getNotifications(data, context) {
  console.log(`Pattern: ${context.getPattern()}`);
}
```

> info **提示** `@Payload()`、`@Ctx()`和`RmqContext`从`@nestjs/microservices`包导入。

要访问原始的RabbitMQ消息（包含`properties`、`fields`和`content`），请使用`RmqContext`对象的`getMessage()`方法，如下所示：

```typescript
@@filename()
@MessagePattern('notifications')
getNotifications(@Payload() data: number[], @Ctx() context: RmqContext) {
  console.log(context.getMessage());
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('notifications')
getNotifications(data, context) {
  console.log(context.getMessage());
}
```

要检索RabbitMQ[通道](https://www.rabbitmq.com/channels.html)的引用，请使用`RmqContext`对象的`getChannelRef`方法，如下所示：

```typescript
@@filename()
@MessagePattern('notifications')
getNotifications(@Payload() data: number[], @Ctx() context: RmqContext) {
  console.log(context.getChannelRef());
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('notifications')
getNotifications(data, context) {
  console.log(context.getChannelRef());
}
```

#### 消息确认

为确保消息永不丢失，RabbitMQ支持[消息确认](https://www.rabbitmq.com/confirms.html)。消费者会发送一个确认信息给RabbitMQ，告知特定消息已被接收、处理，RabbitMQ可以将其删除。如果消费者终止（其通道关闭、连接关闭或TCP连接丢失）而未发送确认，RabbitMQ会认为消息未被完全处理，并将其重新排队。

要启用手动确认模式，请将`noAck`属性设置为`false`：

```typescript
options: {
  urls: ['amqp://localhost:5672'],
  queue: 'cats_queue',
  noAck: false,
  queueOptions: {
    durable: false
  },
},
```

当手动消费者确认模式开启时，我们必须从工作进程发送适当的确认，以表明我们已完成任务。

```typescript
@@filename()
@MessagePattern('notifications')
getNotifications(@Payload() data: number[], @Ctx() context: RmqContext) {
  const channel = context.getChannelRef();
  const originalMsg = context.getMessage();

  channel.ack(originalMsg);
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('notifications')
getNotifications(data, context) {
  const channel = context.getChannelRef();
  const originalMsg = context.getMessage();

  channel.ack(originalMsg);
}
```

#### 记录构建器

要配置消息选项，你可以使用`RmqRecordBuilder`类（注意：这也适用于基于事件的流）。例如，要设置`headers`和`priority`属性，请使用`setOptions`方法，如下所示：

```typescript
const message = ':cat:';
const record = new RmqRecordBuilder(message)
  .setOptions({
    headers: {
      ['x-version']: '1.0.0',
    },
    priority: 3,
  })
  .build();

this.client.send('replace-emoji', record).subscribe(...);
```

> info **提示** `RmqRecordBuilder`类从`@nestjs/microservices`包导出。

你也可以在服务器端读取这些值，通过访问`RmqContext`，如下所示：

```typescript
@@filename()
@MessagePattern('replace-emoji')
replaceEmoji(@Payload() data: string, @Ctx() context: RmqContext): string {
  const { properties: { headers } } = context.getMessage();
  return headers['x-version'] === '1.0.0' ? '🐱' : '🐈';
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('replace-emoji')
replaceEmoji(data, context) {
  const { properties: { headers } } = context.getMessage();
  return headers['x-version'] === '1.0.0' ? '🐱' : '🐈';
}
```

#### 实例状态更新

要获取有关连接和底层驱动实例状态的实时更新，你可以订阅`status`流。此流提供特定于所选驱动的状态更新。对于RMQ驱动，`status`流会发出`connected`和`disconnected`事件。

```typescript
this.client.status.subscribe((status: RmqStatus) => {
  console.log(status);
});
```

> info **提示** `RmqStatus`类型从`@nestjs/microservices`包导入。

同样，你可以订阅服务器的`status`流以接收有关服务器状态的通知。

```typescript
const server = app.connectMicroservice<MicroserviceOptions>(...);
server.status.subscribe((status: RmqStatus) => {
  console.log(status);
});
```

#### 监听RabbitMQ事件

在某些情况下，你可能希望监听微服务发出的内部事件。例如，你可以监听`error`事件，以便在发生错误时触发额外的操作。要做到这一点，请使用`on()`方法，如下所示：

```typescript
this.client.on('error', (err) => {
  console.error(err);
});
```

同样，你可以监听服务器的内部事件：

```typescript
server.on<RmqEvents>('error', (err) => {
  console.error(err);
});
```

> info **提示** `RmqEvents`类型从`@nestjs/microservices`包导入。

#### 底层驱动访问

对于更高级的用例，你可能需要访问底层驱动实例。这在诸如手动关闭连接或使用特定于驱动的方法等场景中可能很有用。但是，请记住，在大多数情况下，你**应该不需要**直接访问驱动。

要做到这一点，你可以使用`unwrap()`方法，该方法返回底层驱动实例。泛型类型参数应指定你期望的驱动实例类型。

```typescript
const managerRef =
  this.client.unwrap<import('amqp-connection-manager').AmqpConnectionManager>();
```

同样，你可以访问服务器的底层驱动实例：

```typescript
const managerRef =
  server.unwrap<import('amqp-connection-manager').AmqpConnectionManager>();
```