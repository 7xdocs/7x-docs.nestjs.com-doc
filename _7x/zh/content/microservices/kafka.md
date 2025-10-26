### Kafka

[Kafka](https://kafka.apache.org/) 是一个开源的分布式流处理平台，它具有三个核心功能：

- 发布和订阅记录流，类似于消息队列或企业消息系统。
- 以容错、持久的方式存储记录流。
- 在记录流产生时对其进行处理。

Kafka 项目旨在提供一个统一的、高吞吐量、低延迟的平台，用于处理实时数据馈送。它与 Apache Storm 和 Spark 集成良好，可用于实时流数据分析。

#### 安装

要开始构建基于 Kafka 的微服务，首先安装所需的包：

```bash
$ npm i --save kafkajs
```

#### 概述

与其他 Nest 微服务传输层实现一样，你可以通过传递给 `createMicroservice()` 方法的选项对象的 `transport` 属性（以及可选的 `options` 属性）来选择 Kafka 传输机制，如下所示：

```typescript
@@filename(main)
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.KAFKA,
  options: {
    client: {
      brokers: ['localhost:9092'],
    }
  }
});
@@switch
const app = await NestFactory.createMicroservice(AppModule, {
  transport: Transport.KAFKA,
  options: {
    client: {
      brokers: ['localhost:9092'],
    }
  }
});
```

> info **提示** `Transport` 枚举从 `@nestjs/microservices` 包导入。

#### 选项

`options` 属性特定于所选的传输器。**Kafka** 传输器公开如下属性。

<table>
  <tr>
    <td><code>client</code></td>
    <td>客户端配置选项（更多信息参见
      <a
        href="https://kafka.js.org/docs/configuration"
        rel="nofollow"
        target="blank"
        >此处</a
      >）</td>
  </tr>
  <tr>
    <td><code>consumer</code></td>
    <td>消费者配置选项（更多信息参见
      <a
        href="https://kafka.js.org/docs/consuming#a-name-options-a-options"
        rel="nofollow"
        target="blank"
        >此处</a
      >）</td>
  </tr>
  <tr>
    <td><code>run</code></td>
    <td>运行配置选项（更多信息参见
      <a
        href="https://kafka.js.org/docs/consuming"
        rel="nofollow"
        target="blank"
        >此处</a
      >）</td>
  </tr>
  <tr>
    <td><code>subscribe</code></td>
    <td>订阅配置选项（更多信息参见
      <a
        href="https://kafka.js.org/docs/consuming#frombeginning"
        rel="nofollow"
        target="blank"
        >此处</a
      >）</td>
  </tr>
  <tr>
    <td><code>producer</code></td>
    <td>生产者配置选项（更多信息参见
      <a
        href="https://kafka.js.org/docs/producing#options"
        rel="nofollow"
        target="blank"
        >此处</a
      >）</td>
  </tr>
  <tr>
    <td><code>send</code></td>
    <td>发送配置选项（更多信息参见
      <a
        href="https://kafka.js.org/docs/producing#options"
        rel="nofollow"
        target="blank"
        >此处</a
      >）</td>
  </tr>
  <tr>
    <td><code>producerOnlyMode</code></td>
    <td>用于跳过消费者组注册，仅作为生产者的功能标志（<code>boolean</code> 类型）</td>
  </tr>
  <tr>
    <td><code>postfixId</code></td>
    <td>更改 clientId 值的后缀（<code>string</code> 类型）</td>
  </tr>
</table>

#### 客户端

Kafka 与其他微服务传输器相比有一个小差异。我们使用 `ClientKafkaProxy` 类，而不是 `ClientProxy` 类。

与其他微服务传输器一样，创建 `ClientKafkaProxy` 实例有 <a href="https://docs.nestjs.com/microservices/basics#client">多种选项</a>。

创建实例的一种方法是使用 `ClientsModule`。要通过 `ClientsModule` 创建客户端实例，导入它并使用 `register()` 方法传递一个选项对象，该对象具有与上面 `createMicroservice()` 方法中所示相同的属性，以及一个用作注入令牌的 `name` 属性。更多关于 `ClientsModule` 的信息参见 <a href="https://docs.nestjs.com/microservices/basics#client">此处</a>。

```typescript
@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'HERO_SERVICE',
        transport: Transport.KAFKA,
        options: {
          client: {
            clientId: 'hero',
            brokers: ['localhost:9092'],
          },
          consumer: {
            groupId: 'hero-consumer'
          }
        }
      },
    ]),
  ]
  ...
})
```

也可以使用其他创建客户端的选项（`ClientProxyFactory` 或 `@Client()`）。你可以在 <a href="https://docs.nestjs.com/microservices/basics#client">此处</a> 了解相关信息。

使用 `@Client()` 装饰器的方式如下：

```typescript
@Client({
  transport: Transport.KAFKA,
  options: {
    client: {
      clientId: 'hero',
      brokers: ['localhost:9092'],
    },
    consumer: {
      groupId: 'hero-consumer'
    }
  }
})
client: ClientKafkaProxy;
```

#### 消息模式

Kafka 微服务消息模式利用两个主题分别作为请求和回复通道。`ClientKafkaProxy#send()` 方法通过将[关联 ID](https://www.enterpriseintegrationpatterns.com/patterns/messaging/CorrelationIdentifier.html)、回复主题和回复分区与请求消息相关联，发送带有[返回地址](https://www.enterpriseintegrationpatterns.com/patterns/messaging/ReturnAddress.html)的消息。这要求 `ClientKafkaProxy` 实例在发送消息之前订阅回复主题并分配到至少一个分区。

因此，每个运行的 Nest 应用程序需要至少有一个回复主题分区。例如，如果你运行 4 个 Nest 应用程序，但回复主题只有 3 个分区，那么其中 1 个 Nest 应用程序在尝试发送消息时会出错。

当新的 `ClientKafkaProxy` 实例启动时，它们会加入消费者组并订阅各自的主题。此过程会触发对分配给消费者组中消费者的主题分区的重新平衡。

通常，主题分区使用轮询分区器进行分配，该分区器将主题分区分配给按消费者名称排序的消费者集合，而消费者名称在应用程序启动时随机设置。但是，当新消费者加入消费者组时，新消费者可以位于消费者集合中的任何位置。这会导致一个情况：当现有消费者位于新消费者之后时，现有消费者可能会被分配不同的分区。结果，被分配不同分区的消费者将丢失在重新平衡之前发送的请求的响应消息。

为了防止 `ClientKafkaProxy` 消费者丢失响应消息，Nest 内置了一个特定的自定义分区器。此自定义分区器将分区分配给按应用程序启动时设置的高分辨率时间戳（`process.hrtime()`）排序的消费者集合。

#### 消息响应订阅

> warning **注意** 本节仅与[请求-响应](/microservices/basics#request-response)消息风格（使用 `@MessagePattern` 装饰器和 `ClientKafkaProxy#send` 方法）相关。对于[基于事件](/microservices/basics#event-based)的通信（`@EventPattern` 装饰器和 `ClientKafkaProxy#emit` 方法），不需要订阅响应主题。

`ClientKafkaProxy` 类提供 `subscribeToResponseOf()` 方法。`subscribeToResponseOf()` 方法接收请求的主题名称作为参数，并将派生的回复主题名称添加到回复主题集合中。在实现消息模式时，此方法是必需的。

```typescript
@@filename(heroes.controller)
onModuleInit() {
  this.client.subscribeToResponseOf('hero.kill.dragon');
}
```

如果 `ClientKafkaProxy` 实例是异步创建的，则必须在调用 `connect()` 方法之前调用 `subscribeToResponseOf()` 方法。

```typescript
@@filename(heroes.controller)
async onModuleInit() {
  this.client.subscribeToResponseOf('hero.kill.dragon');
  await this.client.connect();
}
```

#### 传入消息

Nest 接收传入的 Kafka 消息作为一个包含 `key`、`value` 和 `headers` 属性的对象，这些属性的值类型为 `Buffer`。然后，Nest 通过将缓冲区转换为字符串来解析这些值。如果该字符串“类似对象”，Nest 会尝试将该字符串解析为 `JSON`。然后将 `value` 传递给其关联的处理程序。

#### 传出消息

Nest 在发布事件或发送消息时，会对传递给 `ClientKafkaProxy` 的 `emit()` 和 `send()` 方法的参数，或者从 `@MessagePattern` 方法返回的值进行序列化处理后发送传出的 Kafka 消息。此序列化过程会通过 `JSON.stringify()` 或 `toString()` 原型方法，将非字符串或非缓冲区的对象“字符串化”。

```typescript
@@filename(heroes.controller)
@Controller()
export class HeroesController {
  @MessagePattern('hero.kill.dragon')
  killDragon(@Payload() message: KillDragonMessage): any {
    const dragonId = message.dragonId;
    const items = [
      { id: 1, name: 'Mythical Sword' },
      { id: 2, name: 'Key to Dungeon' },
    ];
    return items;
  }
}
```

> info **提示** `@Payload()` 从 `@nestjs/microservices` 包导入。

也可以通过传递一个包含 `key` 和 `value` 属性的对象来为传出消息设置键。设置消息键对于满足[共分区要求](https://docs.confluent.io/current/ksql/docs/developer-guide/partition-data.html#co-partitioning-requirements)很重要。

```typescript
@@filename(heroes.controller)
@Controller()
export class HeroesController {
  @MessagePattern('hero.kill.dragon')
  killDragon(@Payload() message: KillDragonMessage): any {
    const realm = 'Nest';
    const heroId = message.heroId;
    const dragonId = message.dragonId;

    const items = [
      { id: 1, name: 'Mythical Sword' },
      { id: 2, name: 'Key to Dungeon' },
    ];

    return {
      headers: {
        realm
      },
      key: heroId,
      value: items
    }
  }
}
```

此外，以此格式传递的消息还可以包含在 `headers` 哈希属性中设置的自定义标头。标头哈希属性的值必须是 `string` 类型或 `Buffer` 类型。

```typescript
@@filename(heroes.controller)
@Controller()
export class HeroesController {
  @MessagePattern('hero.kill.dragon')
  killDragon(@Payload() message: KillDragonMessage): any {
    const realm = 'Nest';
    const heroId = message.heroId;
    const dragonId = message.dragonId;

    const items = [
      { id: 1, name: 'Mythical Sword' },
      { id: 2, name: 'Key to Dungeon' },
    ];

    return {
      headers: {
        kafka_nestRealm: realm
      },
      key: heroId,
      value: items
    }
  }
}
```

#### 基于事件

虽然请求-响应方法非常适合服务之间的消息交换，但当你的消息风格是基于事件的（这反过来又非常适合 Kafka）时，它就不太合适了——此时你只想发布事件**而不等待响应**。在这种情况下，你不希望请求-响应机制为维护两个主题而带来额外开销。

查看以下两个部分以了解更多信息：[概述：基于事件](/microservices/basics#event-based) 和 [概述：发布事件](/microservices/basics#publishing-events)。

#### 上下文

在更复杂的场景中，你可能需要访问有关传入请求的附加信息。使用 Kafka 传输器时，你可以访问 `KafkaContext` 对象。

```typescript
@@filename()
@MessagePattern('hero.kill.dragon')
killDragon(@Payload() message: KillDragonMessage, @Ctx() context: KafkaContext) {
  console.log(`Topic: ${context.getTopic()}`);
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('hero.kill.dragon')
killDragon(message, context) {
  console.log(`Topic: ${context.getTopic()}`);
}
```

> info **提示** `@Payload()`、`@Ctx()` 和 `KafkaContext` 从 `@nestjs/microservices` 包导入。

要访问原始的 Kafka `IncomingMessage` 对象，请使用 `KafkaContext` 对象的 `getMessage()` 方法，如下所示：

```typescript
@@filename()
@MessagePattern('hero.kill.dragon')
killDragon(@Payload() message: KillDragonMessage, @Ctx() context: KafkaContext) {
  const originalMessage = context.getMessage();
  const partition = context.getPartition();
  const { headers, timestamp } = originalMessage;
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('hero.kill.dragon')
killDragon(message, context) {
  const originalMessage = context.getMessage();
  const partition = context.getPartition();
  const { headers, timestamp } = originalMessage;
}
```

其中 `IncomingMessage` 符合以下接口：

```typescript
interface IncomingMessage {
  topic: string;
  partition: number;
  timestamp: string;
  size: number;
  attributes: number;
  offset: string;
  key: any;
  value: any;
  headers: Record<string, any>;
}
```

如果你的处理程序对每个接收的消息涉及较长的处理时间，你应该考虑使用 `heartbeat` 回调。要获取 `heartbeat` 函数，请使用 `KafkaContext` 的 `getHeartbeat()` 方法，如下所示：

```typescript
@@filename()
@MessagePattern('hero.kill.dragon')
async killDragon(@Payload() message: KillDragonMessage, @Ctx() context: KafkaContext) {
  const heartbeat = context.getHeartbeat();

  // Do some slow processing
  await doWorkPart1();

  // Send heartbeat to not exceed the sessionTimeout
  await heartbeat();

  // Do some slow processing again
  await doWorkPart2();
}
```

#### 命名约定

Kafka 微服务组件会在 `client.clientId` 和 `consumer.groupId` 选项后附加其各自角色的描述，以防止 Nest 微服务客户端和服务器组件之间发生冲突。默认情况下，`ClientKafkaProxy` 组件会在这两个选项后附加 `-client`，而 `ServerKafka` 组件会附加 `-server`。注意下面提供的值是如何按此方式转换的（如注释所示）。

```typescript
@@filename(main)
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.KAFKA,
  options: {
    client: {
      clientId: 'hero', // hero-server
      brokers: ['localhost:9092'],
    },
    consumer: {
      groupId: 'hero-consumer' // hero-consumer-server
    },
  }
});
```

对于客户端：

```typescript
@@filename(heroes.controller)
@Client({
  transport: Transport.KAFKA,
  options: {
    client: {
      clientId: 'hero', // hero-client
      brokers: ['localhost:9092'],
    },
    consumer: {
      groupId: 'hero-consumer' // hero-consumer-client
    }
  }
})
client: ClientKafkaProxy;
```

> info **提示** 可以通过在自定义提供程序中扩展 `ClientKafkaProxy` 和 `KafkaServer` 并覆盖构造函数来自定义 Kafka 客户端和消费者的命名约定。

由于 Kafka 微服务消息模式利用两个主题作为请求和回复通道，回复模式应从请求主题派生。默认情况下，回复主题的名称是请求主题名称加上 `.reply` 组成的。

```typescript
@@filename(heroes.controller)
onModuleInit() {
  this.client.subscribeToResponseOf('hero.get'); // hero.get.reply
}
```

> info **提示** 可以通过在自定义提供程序中扩展 `ClientKafkaProxy` 并覆盖 `getResponsePatternName` 方法来自定义 Kafka 回复主题的命名约定。

#### 可重试异常

与其他传输器类似，所有未处理的异常都会自动包装到 `RpcException` 中，并转换为“用户友好”的格式。但是，在某些边缘情况下，你可能希望绕过此机制，让异常由 `kafkajs` 驱动程序处理。在处理消息时抛出异常会指示 `kafkajs` 对其进行**重试**（重新传递），这意味着即使消息（或事件）处理程序已触发，偏移量也不会提交到 Kafka。

> warning **警告** 对于事件处理程序（基于事件的通信），默认情况下所有未处理的异常都被视为**可重试异常**。

为此，你可以使用一个专门的类 `KafkaRetriableException`，如下所示：

```typescript
throw new KafkaRetriableException('...');
```

> info **提示** `KafkaRetriableException` 类从 `@nestjs/microservices` 包导出。

#### 提交偏移量

在使用 Kafka 时，提交偏移量至关重要。默认情况下，消息会在特定时间后自动提交。有关更多信息，请访问 [KafkaJS 文档](https://kafka.js.org/docs/consuming#autocommit)。`KafkaContext` 提供了一种访问活动消费者以手动提交偏移量的方式。该消费者是 KafkaJS 消费者，其工作方式与 [原生 KafkaJS 实现](https://kafka.js.org/docs/consuming#manual-committing) 一致。

```typescript
@@filename()
@EventPattern('user.created')
async handleUserCreated(@Payload() data: IncomingMessage, @Ctx() context: KafkaContext) {
  // business logic

  const { offset } = context.getMessage();
  const partition = context.getPartition();
  const topic = context.getTopic();
  const consumer = context.getConsumer();
  await consumer.commitOffsets([{ topic, partition, offset }])
}
@@switch
@Bind(Payload(), Ctx())
@EventPattern('user.created')
async handleUserCreated(data, context) {
  // business logic

  const { offset } = context.getMessage();
  const partition = context.getPartition();
  const topic = context.getTopic();
  const consumer = context.getConsumer();
  await consumer.commitOffsets([{ topic, partition, offset }])
}
```

要禁用消息的自动提交，请在 `run` 配置中设置 `autoCommit: false`，如下所示：

```typescript
@@filename(main)
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.KAFKA,
  options: {
    client: {
      brokers: ['localhost:9092'],
    },
    run: {
      autoCommit: false
    }
  }
});
@@switch
const app = await NestFactory.createMicroservice(AppModule, {
  transport: Transport.KAFKA,
  options: {
    client: {
      brokers: ['localhost:9092'],
    },
    run: {
      autoCommit: false
    }
  }
});
```

#### 实例状态更新

要获取关于连接和底层驱动程序实例状态的实时更新，你可以订阅 `status` 流。此流提供特定于所选驱动程序的状态更新。对于 Kafka 驱动程序，`status` 流会发出 `connected`、`disconnected`、`rebalancing`、`crashed` 和 `stopped` 事件。

```typescript
this.client.status.subscribe((status: KafkaStatus) => {
  console.log(status);
});
```

> info **提示** `KafkaStatus` 类型从 `@nestjs/microservices` 包导入。

同样，你可以订阅服务器的 `status` 流以接收有关服务器状态的通知。

```typescript
const server = app.connectMicroservice<MicroserviceOptions>(...);
server.status.subscribe((status: KafkaStatus) => {
  console.log(status);
});
```

#### 底层生产者和消费者

对于更高级的用例，你可能需要访问底层的生产者和消费者实例。这在诸如手动关闭连接或使用特定于驱动程序的方法等场景中可能很有用。但是，请记住，在大多数情况下，你**应该不需要**直接访问驱动程序。

为此，你可以使用 `ClientKafkaProxy` 实例公开的 `producer` 和 `consumer`  getter 方法。

```typescript
const producer = this.client.producer;
const consumer = this.client.consumer;
```