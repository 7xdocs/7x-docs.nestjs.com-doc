### NATS

[NATS](https://nats.io)是一个简单、安全且高性能的开源消息系统，适用于云原生应用、物联网消息传递和微服务架构。NATS服务器使用Go编程语言编写，但用于与服务器交互的客户端库支持数十种主要编程语言。NATS支持**最多一次**和**至少一次**的消息传递方式。它可以在任何地方运行，从大型服务器和云实例，到边缘网关，甚至物联网设备。

#### 安装

要开始构建基于NATS的微服务，首先安装所需的包：

```bash
$ npm i --save nats
```

#### 概述

要使用NATS传输器，请将以下选项对象传递给`createMicroservice()`方法：

```typescript
@@filename(main)
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.NATS,
  options: {
    servers: ['nats://localhost:4222'],
  },
});
@@switch
const app = await NestFactory.createMicroservice(AppModule, {
  transport: Transport.NATS,
  options: {
    servers: ['nats://localhost:4222'],
  },
});
```

> info **提示** `Transport`枚举从`@nestjs/microservices`包导入。

#### 选项

`options`对象特定于所选的传输器。**NATS**传输器公开了[此处](https://github.com/nats-io/node-nats#connection-options)描述的属性以及以下属性：

<table>
  <tr>
    <td><code>queue</code></td>
    <td>服务器应订阅的队列（留空<code>undefined</code>可忽略此设置）。在下方了解更多关于NATS队列组的信息。
    </td> 
  </tr>
  <tr>
    <td><code>gracefulShutdown</code></td>
    <td>启用优雅关闭。启用后，服务器在关闭连接之前会先从所有通道取消订阅。默认值为<code>false</code>。
  </tr>
  <tr>
    <td><code>gracePeriod</code></td>
    <td>从所有通道取消订阅后等待服务器的时间（以毫秒为单位）。默认值为<code>10000</code>毫秒。
  </tr>
</table>

#### 客户端

与其他微服务传输器一样，创建NATS`ClientProxy`实例有<a href="https://docs.nestjs.com/microservices/basics#client">多种选项</a>。

创建实例的一种方法是使用`ClientsModule`。要使用`ClientsModule`创建客户端实例，请导入它并使用`register()`方法传递一个选项对象，该对象具有与上面`createMicroservice()`方法中所示相同的属性，以及一个用作注入令牌的`name`属性。在<a href="https://docs.nestjs.com/microservices/basics#client">此处</a>了解更多关于`ClientsModule`的信息。

```typescript
@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'MATH_SERVICE',
        transport: Transport.NATS,
        options: {
          servers: ['nats://localhost:4222'],
        }
      },
    ]),
  ]
  ...
})
```

也可以使用其他创建客户端的选项（`ClientProxyFactory`或`@Client()`）。你可以在<a href="https://docs.nestjs.com/microservices/basics#client">此处</a>了解它们。

#### 请求-响应

对于**请求-响应**消息样式（[了解更多](https://docs.nestjs.com/microservices/basics#request-response)），NATS传输器不使用NATS内置的[请求-回复](https://docs.nats.io/nats-concepts/reqreply)机制。相反，“请求”通过`publish()`方法在给定主题上发布，并带有唯一的回复主题名称，响应者监听该主题并将响应发送到回复主题。回复主题会动态定向回请求者，无论双方的位置如何。

#### 基于事件

对于**基于事件**的消息样式（[了解更多](https://docs.nestjs.com/microservices/basics#event-based)），NATS传输器使用NATS内置的[发布-订阅](https://docs.nats.io/nats-concepts/pubsub)机制。发布者在某个主题上发送消息，任何正在监听该主题的活动订阅者都会收到该消息。订阅者还可以注册对通配符主题的兴趣，其工作方式类似于正则表达式。这种一对多模式有时称为扇出。

#### 队列组

NATS提供了一个称为[分布式队列](https://docs.nats.io/nats-concepts/queue)的内置负载均衡功能。要创建队列订阅，请使用`queue`属性，如下所示：

```typescript
@@filename(main)
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.NATS,
  options: {
    servers: ['nats://localhost:4222'],
    queue: 'cats_queue',
  },
});
```

#### 上下文

在更复杂的场景中，你可能需要访问有关传入请求的附加信息。使用NATS传输器时，可以访问`NatsContext`对象。

```typescript
@@filename()
@MessagePattern('notifications')
getNotifications(@Payload() data: number[], @Ctx() context: NatsContext) {
  console.log(`Subject: ${context.getSubject()}`);
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('notifications')
getNotifications(data, context) {
  console.log(`Subject: ${context.getSubject()}`);
}
```

> info **提示** `@Payload()`、`@Ctx()`和`NatsContext`从`@nestjs/microservices`包导入。

#### 通配符

订阅可以是针对明确的主题，也可以包含通配符。

```typescript
@@filename()
@MessagePattern('time.us.*')
getDate(@Payload() data: number[], @Ctx() context: NatsContext) {
  console.log(`Subject: ${context.getSubject()}`); // 例如 "time.us.east"
  return new Date().toLocaleTimeString(...);
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('time.us.*')
getDate(data, context) {
  console.log(`Subject: ${context.getSubject()}`); // 例如 "time.us.east"
  return new Date().toLocaleTimeString(...);
}
```

#### 记录构建器

要配置消息选项，可以使用`NatsRecordBuilder`类（注意：这也适用于基于事件的流）。例如，要添加`x-version`标头，请使用`setHeaders`方法，如下所示：

```typescript
import * as nats from 'nats';

// 在代码中的某处
const headers = nats.headers();
headers.set('x-version', '1.0.0');

const record = new NatsRecordBuilder(':cat:').setHeaders(headers).build();
this.client.send('replace-emoji', record).subscribe(...);
```

> info **提示** `NatsRecordBuilder`类从`@nestjs/microservices`包导出。

你也可以在服务器端通过访问`NatsContext`来读取这些标头，如下所示：

```typescript
@@filename()
@MessagePattern('replace-emoji')
replaceEmoji(@Payload() data: string, @Ctx() context: NatsContext): string {
  const headers = context.getHeaders();
  return headers['x-version'] === '1.0.0' ? '🐱' : '🐈';
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('replace-emoji')
replaceEmoji(data, context) {
  const headers = context.getHeaders();
  return headers['x-version'] === '1.0.0' ? '🐱' : '🐈';
}
```

在某些情况下，你可能希望为多个请求配置标头，可以将这些标头作为选项传递给`ClientProxyFactory`：

```typescript
import { Module } from '@nestjs/common';
import { ClientProxyFactory, Transport } from '@nestjs/microservices';

@Module({
  providers: [
    {
      provide: 'API_v1',
      useFactory: () =>
        ClientProxyFactory.create({
          transport: Transport.NATS,
          options: {
            servers: ['nats://localhost:4222'],
            headers: { 'x-version': '1.0.0' },
          },
        }),
    },
  ],
})
export class ApiModule {}
```

#### 实例状态更新

要获取有关连接和底层驱动程序实例状态的实时更新，可以订阅`status`流。此流提供特定于所选驱动程序的状态更新。对于NATS驱动程序，`status`流会发出`connected`、`disconnected`和`reconnecting`事件。

```typescript
this.client.status.subscribe((status: NatsStatus) => {
  console.log(status);
});
```

> info **提示** `NatsStatus`类型从`@nestjs/microservices`包导入。

同样，你可以订阅服务器的`status`流以接收有关服务器状态的通知。

```typescript
const server = app.connectMicroservice<MicroserviceOptions>(...);
server.status.subscribe((status: NatsStatus) => {
  console.log(status);
});
```

#### 监听Nats事件

在某些情况下，你可能希望监听微服务发出的内部事件。例如，你可以监听`error`事件，以便在发生错误时触发额外的操作。要做到这一点，请使用`on()`方法，如下所示：

```typescript
this.client.on('error', (err) => {
  console.error(err);
});
```

同样，你可以监听服务器的内部事件：

```typescript
server.on<NatsEvents>('error', (err) => {
  console.error(err);
});
```

> info **提示** `NatsEvents`类型从`@nestjs/microservices`包导入。

#### 底层驱动程序访问

对于更高级的用例，你可能需要访问底层驱动程序实例。这在诸如手动关闭连接或使用驱动程序特定方法等场景中很有用。但是，请记住，在大多数情况下，你**不需要**直接访问驱动程序。

要访问底层驱动程序实例，可以使用`unwrap()`方法，该方法返回底层驱动程序实例。泛型类型参数应指定你期望的驱动程序实例类型。

```typescript
const natsConnection = this.client.unwrap<import('nats').NatsConnection>();
```

同样，你可以访问服务器的底层驱动程序实例：

```typescript
const natsConnection = server.unwrap<import('nats').NatsConnection>();
```