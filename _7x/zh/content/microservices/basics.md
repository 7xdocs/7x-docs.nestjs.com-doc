### 概述

除了传统（有时称为单体）应用架构外，Nest原生支持微服务架构风格的开发。本文档其他部分讨论的大多数概念，如依赖注入、装饰器、异常过滤器、管道、守卫和拦截器，同样适用于微服务。在可能的情况下，Nest会抽象实现细节，以便相同的组件可以在基于HTTP的平台、WebSocket和微服务之间运行。本节介绍Nest中特定于微服务的方面。

在Nest中，微服务本质上是一个使用与HTTP不同的**传输**层的应用程序。

<figure><img class="illustrative-image" src="/assets/Microservices_1.png" /></figure>

Nest支持几种内置的传输层实现，称为**传输器**，它们负责在不同的微服务实例之间传输消息。大多数传输器原生支持**请求-响应**和**基于事件**的消息风格。Nest在请求-响应和基于事件的消息传递的规范接口后抽象了每个传输器的实现细节。这使得从一个传输层切换到另一个传输层变得容易——例如，为了利用特定传输层的特定可靠性或性能特性——而不会影响你的应用程序代码。

#### 安装

要开始构建微服务，首先安装所需的包：

```bash
$ npm i --save @nestjs/microservices
```

#### 入门

要实例化一个微服务，请使用`NestFactory`类的`createMicroservice()`方法：

```typescript
@@filename(main)
import { NestFactory } from '@nestjs/core';
import { Transport, MicroserviceOptions } from '@nestjs/microservices';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.createMicroservice<MicroserviceOptions>(
    AppModule,
    {
      transport: Transport.TCP,
    },
  );
  await app.listen();
}
bootstrap();
@@switch
import { NestFactory } from '@nestjs/core';
import { Transport } from '@nestjs/microservices';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.createMicroservice(AppModule, {
    transport: Transport.TCP,
  });
  await app.listen();
}
bootstrap();
```

> info **提示** 微服务默认使用**TCP**传输层。

`createMicroservice()`方法的第二个参数是一个`options`对象。该对象可能包含两个成员：

<table>
  <tr>
    <td><code>transport</code></td>
    <td>指定传输器（例如，<code>Transport.NATS</code>）</td>
  </tr>
  <tr>
    <td><code>options</code></td>
    <td>特定于传输器的选项对象，用于确定传输器行为</td>
  </tr>
</table>
<p>
  <code>options</code>对象特定于所选的传输器。<strong>TCP</strong>传输器公开
  如下所述的属性。对于其他传输器（例如，Redis、MQTT等），请参阅相关章节以了解可用选项的描述。
</p>
<table>
  <tr>
    <td><code>host</code></td>
    <td>连接主机名</td>
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
    <td><code>serializer</code></td>
    <td>用于传出消息的自定义<a href="https://github.com/nestjs/nest/blob/master/packages/microservices/interfaces/serializer.interface.ts" target="_blank">序列化器</a></td>
  </tr>
  <tr>
    <td><code>deserializer</code></td>
    <td>用于传入消息的自定义<a href="https://github.com/nestjs/nest/blob/master/packages/microservices/interfaces/deserializer.interface.ts" target="_blank">反序列化器</a></td>
  </tr>
  <tr>
    <td><code>socketClass</code></td>
    <td>扩展<code>TcpSocket</code>的自定义Socket（默认：<code>JsonSocket</code>）</td>
  </tr>
  <tr>
    <td><code>tlsOptions</code></td>
    <td>用于配置tls协议的选项</td>
  </tr>
</table>

> info **提示** 上述属性特定于TCP传输器。有关其他传输器的可用选项信息，请参阅相关章节。

#### 消息和事件模式

微服务通过**模式**识别消息和事件。模式是一个普通值，例如，一个文字对象或一个字符串。模式会自动序列化并与消息的数据部分一起通过网络发送。通过这种方式，消息发送者和消费者可以协调哪些请求由哪些处理程序消费。

#### 请求-响应

当你需要在各种外部服务之间**交换**消息时，请求-响应消息风格非常有用。这种范式确保服务实际上已收到消息（无需你手动实现确认协议）。然而，请求-响应方法可能并不总是最佳选择。例如，流式传输器，如[Kafka](https://docs.confluent.io/3.0.0/streams/)或[NATS streaming](https://github.com/nats-io/node-nats-streaming)，它们使用基于日志的持久性，针对解决一系列不同的挑战进行了优化，更符合事件消息传递范式（详见[基于事件的消息传递](https://docs.nestjs.com/microservices/basics#event-based)）。

为了支持请求-响应消息类型，Nest创建了两个逻辑通道：一个用于传输数据，另一个用于等待传入的响应。对于某些底层传输，如[NATS](https://nats.io/)，这种双通道支持是开箱即用的。对于其他传输，Nest通过手动创建单独的通道来补偿。虽然这很有效，但可能会引入一些开销。因此，如果你不需要请求-响应消息风格，你可能需要考虑使用基于事件的方法。

要创建基于请求-响应范式的消息处理程序，请使用`@MessagePattern()`装饰器，该装饰器从`@nestjs/microservices`包导入。此装饰器应仅在[控制器](https://docs.nestjs.com/controllers)类中使用，因为它们充当应用程序的入口点。在提供者中使用它将没有效果，因为它们会被Nest运行时忽略。

```typescript
@@filename(math.controller)
import { Controller } from '@nestjs/common';
import { MessagePattern } from '@nestjs/microservices';

@Controller()
export class MathController {
  @MessagePattern({ cmd: 'sum' })
  accumulate(data: number[]): number {
    return (data || []).reduce((a, b) => a + b);
  }
}
@@switch
import { Controller } from '@nestjs/common';
import { MessagePattern } from '@nestjs/microservices';

@Controller()
export class MathController {
  @MessagePattern({ cmd: 'sum' })
  accumulate(data) {
    return (data || []).reduce((a, b) => a + b);
  }
}
```

在上面的代码中，`accumulate()`**消息处理程序**监听与`{{ '{' }} cmd: 'sum' {{ '}' }}`消息模式匹配的消息。消息处理程序接受一个参数，即从客户端传递的`data`。在这种情况下，数据是需要累加的数字数组。

#### 异步响应

消息处理程序可以同步或**异步**响应，这意味着支持`async`方法。

```typescript
@@filename()
@MessagePattern({ cmd: 'sum' })
async accumulate(data: number[]): Promise<number> {
  return (data || []).reduce((a, b) => a + b);
}
@@switch
@MessagePattern({ cmd: 'sum' })
async accumulate(data) {
  return (data || []).reduce((a, b) => a + b);
}
```

消息处理程序也可以返回`Observable`，在这种情况下，结果值将被发射，直到流完成。

```typescript
@@filename()
@MessagePattern({ cmd: 'sum' })
accumulate(data: number[]): Observable<number> {
  return from([1, 2, 3]);
}
@@switch
@MessagePattern({ cmd: 'sum' })
accumulate(data: number[]): Observable<number> {
  return from([1, 2, 3]);
}
```

在上面的示例中，消息处理程序将**响应三次**，数组中的每个项目一次。

#### 基于事件的

虽然请求-响应方法非常适合在服务之间交换消息，但它不太适合基于事件的消息传递——当你只想发布**事件**而不等待响应时。在这种情况下，为请求-响应维护两个通道的开销是不必要的。

例如，如果你想通知另一个服务系统的这一部分发生了特定情况，基于事件的消息风格是理想的。

要创建事件处理程序，你可以使用`@EventPattern()`装饰器，该装饰器从`@nestjs/microservices`包导入。

```typescript
@@filename()
@EventPattern('user_created')
async handleUserCreated(data: Record<string, unknown>) {
  // business logic
}
@@switch
@EventPattern('user_created')
async handleUserCreated(data) {
  // business logic
}
```

> info **提示** 你可以为**单个**事件模式注册多个事件处理程序，所有这些处理程序都将自动并行触发。

`handleUserCreated()`**事件处理程序**监听`'user_created'`事件。事件处理程序接受一个参数，即从客户端传递的`data`（在这种情况下，是通过网络发送的事件负载）。

<app-banner-enterprise></app-banner-enterprise>

#### 额外的请求详情

在更复杂的场景中，你可能需要访问有关传入请求的额外详情。例如，当使用带有通配符订阅的NATS时，你可能希望检索生产者发送消息到的原始主题。同样，对于Kafka，你可能需要访问消息头。要实现这一点，你可以如下所示利用内置装饰器：

```typescript
@@filename()
@MessagePattern('time.us.*')
getDate(@Payload() data: number[], @Ctx() context: NatsContext) {
  console.log(`Subject: ${context.getSubject()}`); // e.g. "time.us.east"
  return new Date().toLocaleTimeString(...);
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('time.us.*')
getDate(data, context) {
  console.log(`Subject: ${context.getSubject()}`); // e.g. "time.us.east"
  return new Date().toLocaleTimeString(...);
}
```

> info **提示** `@Payload()`、`@Ctx()`和`NatsContext`从`@nestjs/microservices`导入。

> info **提示** 你也可以将属性键传递给`@Payload()`装饰器，以从传入的负载对象中提取特定属性，例如，`@Payload('id')`。

#### 客户端（生产者类）

客户端Nest应用程序可以使用`ClientProxy`类与Nest微服务交换消息或发布事件。此类提供了几种方法，如`send()`（用于请求-响应消息传递）和`emit()`（用于事件驱动消息传递），支持与远程微服务通信。你可以通过以下方式获取此类的实例：

一种方法是导入`ClientsModule`，它公开了静态`register()`方法。此方法接受一组表示微服务传输器的对象。每个对象必须包含一个`name`属性，可选的`transport`属性（默认为`Transport.TCP`），以及可选的`options`属性。

`name`属性充当**注入令牌**，你可以使用它在任何需要的地方注入`ClientProxy`的实例。`name`属性的值可以是任何任意字符串或JavaScript符号，如[此处](https://docs.nestjs.com/fundamentals/custom-providers#non-class-based-provider-tokens)所述。

`options`属性是一个对象，包含我们之前在`createMicroservice()`方法中看到的相同属性。

```typescript
@Module({
  imports: [
    ClientsModule.register([
      { name: 'MATH_SERVICE', transport: Transport.TCP },
    ]),
  ],
})
```

或者，如果你需要在设置期间提供配置或执行任何其他异步过程，你可以使用`registerAsync()`方法。

```typescript
@Module({
  imports: [
    ClientsModule.registerAsync([
      {
        imports: [ConfigModule],
        name: 'MATH_SERVICE',
        useFactory: async (configService: ConfigService) => ({
          transport: Transport.TCP,
          options: {
            url: configService.get('URL'),
          },
        }),
        inject: [ConfigService],
      },
    ]),
  ],
})
```

导入模块后，你可以使用`@Inject()`装饰器注入为`'MATH_SERVICE'`传输器配置了指定选项的`ClientProxy`实例。

```typescript
constructor(
  @Inject('MATH_SERVICE') private client: ClientProxy,
) {}
```

> info **提示** `ClientsModule`和`ClientProxy`类从`@nestjs/microservices`包导入。

有时，你可能需要从另一个服务（如`ConfigService`）获取传输器配置，而不是在客户端应用程序中硬编码它。要实现这一点，你可以使用`ClientProxyFactory`类注册一个[自定义提供者](/fundamentals/custom-providers)。此类提供了一个静态`create()`方法，该方法接受一个传输器选项对象并返回一个自定义的`ClientProxy`实例。

```typescript
@Module({
  providers: [
    {
      provide: 'MATH_SERVICE',
      useFactory: (configService: ConfigService) => {
        const mathSvcOptions = configService.getMathSvcOptions();
        return ClientProxyFactory.create(mathSvcOptions);
      },
      inject: [ConfigService],
    }
  ]
  ...
})
```

> info **提示** `ClientProxyFactory`从`@nestjs/microservices`包导入。

另一个选项是使用`@Client()`属性装饰器。

```typescript
@Client({ transport: Transport.TCP })
client: ClientProxy;
```

> info **提示** `@Client()`装饰器从`@nestjs/microservices`包导入。

使用`@Client()`装饰器不是首选技术，因为它更难测试且更难共享客户端实例。

`ClientProxy`是**惰性的**。它不会立即启动连接。相反，它将在第一次微服务调用之前建立，然后在每次后续调用中重用。但是，如果你想将应用程序引导过程延迟到建立连接为止，你可以在`OnApplicationBootstrap`生命周期钩子中使用`ClientProxy`对象的`connect()`方法手动启动连接。

```typescript
@@filename()
async onApplicationBootstrap() {
  await this.client.connect();
}
```

如果无法创建连接，`connect()`方法将拒绝并返回相应的错误对象。

#### 发送消息

`ClientProxy`公开了一个`send()`方法。此方法旨在调用微服务并返回带有其响应的`Observable`。因此，我们可以轻松地订阅发射的值。

```typescript
@@filename()
accumulate(): Observable<number> {
  const pattern = { cmd: 'sum' };
  const payload = [1, 2, 3];
  return this.client.send<number>(pattern, payload);
}
@@switch
accumulate() {
  const pattern = { cmd: 'sum' };
  const payload = [1, 2, 3];
  return this.client.send(pattern, payload);
}
```

`send()`方法接受两个参数：`pattern`和`payload`。`pattern`应该与`@MessagePattern()`装饰器中定义的模式匹配。`payload`是我们想要传输到远程微服务的消息。此方法返回一个**冷`Observable`**，这意味着你必须显式订阅它，消息才会被发送。

#### 发布事件

要发送事件，请使用`ClientProxy`对象的`emit()`方法。此方法将事件发布到消息代理。

```typescript
@@filename()
async publish() {
  this.client.emit<number>('user_created', new UserCreatedEvent());
}
@@switch
async publish() {
  this.client.emit('user_created', new UserCreatedEvent());
}
```

`emit()`方法接受两个参数：`pattern`和`payload`。`pattern`应该与`@EventPattern()`装饰器中定义的模式匹配，而`payload`表示你想要传输到远程微服务的事件数据。此方法返回一个**热`Observable`**（与`send()`返回的冷`Observable`相反），这意味着无论你是否显式订阅该可观察对象，代理都将立即尝试传递事件。

<app-banner-devtools></app-banner-devtools>

#### 请求作用域

对于来自不同编程语言背景的人来说，可能会惊讶地发现，在Nest中，大多数东西都是跨传入请求共享的。这包括数据库连接池、具有全局状态的单例服务等。请记住，Node.js不遵循请求/响应多线程无状态模型，其中每个请求由单独的线程处理。因此，使用单例实例对于我们的应用程序是**安全的**。

然而，在某些边缘情况下，处理程序的基于请求的生命周期可能是理想的。这可能包括GraphQL应用程序中的每个请求缓存、请求跟踪或多租户等场景。你可以在[此处](/fundamentals/injection-scopes)了解更多关于如何控制作用域的信息。

请求作用域的处理程序和提供者可以使用`@Inject()`装饰器结合`CONTEXT`令牌注入`RequestContext`：

```typescript
import { Injectable, Scope, Inject } from '@nestjs/common';
import { CONTEXT, RequestContext } from '@nestjs/microservices';

@Injectable({ scope: Scope.REQUEST })
export class CatsService {
  constructor(@Inject(CONTEXT) private ctx: RequestContext) {}
}
```

这提供了对`RequestContext`对象的访问，该对象有两个属性：

```typescript
export interface RequestContext<T = any> {
  pattern: string | Record<string, any>;
  data: T;
}
```

`data`属性是消息生产者发送的消息负载。`pattern`属性是用于识别处理传入消息的适当处理程序的模式。

#### 实例状态更新

要获取关于连接和底层驱动实例状态的实时更新，你可以订阅`status`流。此流提供特定于所选驱动的状态更新。例如，如果你使用TCP传输器（默认），`status`流会发射`connected`和`disconnected`事件。

```typescript
this.client.status.subscribe((status: TcpStatus) => {
  console.log(status);
});
```

> info **提示** `TcpStatus`类型从`@nestjs/microservices`包导入。

同样，你可以订阅服务器的`status`流以接收关于服务器状态的通知。

```typescript
const server = app.connectMicroservice<MicroserviceOptions>(...);
server.status.subscribe((status: TcpStatus) => {
  console.log(status);
});
```

#### 监听内部事件

在某些情况下，你可能想要监听微服务发射的内部事件。例如，你可以监听`error`事件，以便在发生错误时触发额外的操作。要做到这一点，请使用`on()`方法，如下所示：

```typescript
this.client.on('error', (err) => {
  console.error(err);
});
```

同样，你可以监听服务器的内部事件：

```typescript
server.on<TcpEvents>('error', (err) => {
  console.error(err);
});
```

> info **提示** `TcpEvents`类型从`@nestjs/microservices`包导入。

#### 底层驱动访问

对于更高级的用例，你可能需要访问底层驱动实例。这对于诸如手动关闭连接或使用特定于驱动的方法等场景很有用。然而，请记住，在大多数情况下，你**应该不需要**直接访问驱动。

要做到这一点，你可以使用`unwrap()`方法，该方法返回底层驱动实例。泛型类型参数应该指定你期望的驱动实例的类型。

```typescript
const netServer = this.client.unwrap<Server>();
```

这里，`Server`是从`net`模块导入的类型。

同样，你可以访问服务器的底层驱动实例：

```typescript
const netServer = server.unwrap<Server>();
```

#### 处理超时

在分布式系统中，微服务有时可能会宕机或不可用。为了防止无限期等待，你可以使用超时。在与其他服务通信时，超时是一种非常有用的模式。要将超时应用于你的微服务调用，你可以使用[RxJS](https://rxjs.dev)的`timeout`操作符。如果微服务在指定时间内没有响应，将抛出异常，你可以捕获并适当地处理该异常。

要实现这一点，你需要使用[`rxjs`](https://github.com/ReactiveX/rxjs)包。只需在管道中使用`timeout`操作符：

```typescript
@@filename()
this.client
  .send<TResult, TInput>(pattern, data)
  .pipe(timeout(5000));
@@switch
this.client
  .send(pattern, data)
  .pipe(timeout(5000));
```

> info **提示** `timeout`操作符从`rxjs/operators`包导入。

5秒后，如果微服务没有响应，它将抛出错误。

#### TLS支持

在私有网络外通信时，加密流量以确保安全性非常重要。在NestJS中，这可以通过使用Node的内置[TLS](https://nodejs.org/api/tls.html)模块在TCP上使用TLS来实现。Nest在其TCP传输中提供了对TLS的内置支持，允许我们加密微服务或客户端之间的通信。

要为TCP服务器启用TLS，你需要PEM格式的私钥和证书。通过设置`tlsOptions`并指定密钥和证书文件，将这些添加到服务器的选项中，如下所示：

```typescript
import * as fs from 'fs';
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { MicroserviceOptions, Transport } from '@nestjs/microservices';

async function bootstrap() {
  const key = fs.readFileSync('<pathToKeyFile>', 'utf8').toString();
  const cert = fs.readFileSync('<pathToCertFile>', 'utf8').toString();

  const app = await NestFactory.createMicroservice<MicroserviceOptions>(
    AppModule,
    {
      transport: Transport.TCP,
      options: {
        tlsOptions: {
          key,
          cert,
        },
      },
    },
  );

  await app.listen();
}
bootstrap();
```

对于要通过TLS安全通信的客户端，我们也定义`tlsOptions`对象，但这次使用CA证书。这是签署服务器证书的机构的证书。这确保客户端信任服务器的证书并能建立安全连接。

```typescript
import { Module } from '@nestjs/common';
import { ClientsModule, Transport } from '@nestjs/microservices';

@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'MATH_SERVICE',
        transport: Transport.TCP,
        options: {
          tlsOptions: {
            ca: [fs.readFileSync('<pathToCaFile>', 'utf-8').toString()],
          },
        },
      },
    ]),
  ],
})
export class AppModule {}
```

如果你的设置涉及多个受信任的机构，你也可以传递CA数组。

一切设置完成后，你可以像往常一样使用`@Inject()`装饰器注入`ClientProxy`，在你的服务中使用客户端。这确保在你的NestJS微服务之间进行加密通信，由Node的`TLS`模块处理加密细节。

有关更多信息，请参阅Node的[TLS文档](https://nodejs.org/api/tls.html)。