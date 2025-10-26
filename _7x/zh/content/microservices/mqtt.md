### MQTT

[MQTT](https://mqtt.org/)（消息队列遥测传输协议）是一种开源的轻量级消息协议，专为低延迟优化。该协议使用**发布/订阅**模型，提供了一种可扩展且经济高效的设备连接方式。基于MQTT构建的通信系统由发布服务器、代理（broker）和一个或多个客户端组成。它专为受限设备以及低带宽、高延迟或不可靠的网络而设计。

#### 安装

要开始构建基于MQTT的微服务，首先安装所需的包：

```bash
$ npm i --save mqtt
```

#### 概述

要使用MQTT传输器，请将以下选项对象传递给`createMicroservice()`方法：

```typescript
@@filename(main)
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.MQTT,
  options: {
    url: 'mqtt://localhost:1883',
  },
});
@@switch
const app = await NestFactory.createMicroservice(AppModule, {
  transport: Transport.MQTT,
  options: {
    url: 'mqtt://localhost:1883',
  },
});
```

> info **提示** `Transport`枚举从`@nestjs/microservices`包导入。

#### 选项

`options`对象特定于所选的传输器。**MQTT**传输器公开了[此处](https://github.com/mqttjs/MQTT.js/#mqttclientstreambuilder-options)描述的属性。

#### 客户端

与其他微服务传输器一样，创建MQTT`ClientProxy`实例有<a href="https://docs.nestjs.com/microservices/basics#client">多种选项</a>。

创建实例的一种方法是使用`ClientsModule`。要通过`ClientsModule`创建客户端实例，请导入它并使用`register()`方法传递一个选项对象，该对象包含与上面`createMicroservice()`方法中所示相同的属性，以及一个用作注入令牌的`name`属性。有关`ClientsModule`的更多信息，请参见<a href="https://docs.nestjs.com/microservices/basics#client">此处</a>。

```typescript
@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'MATH_SERVICE',
        transport: Transport.MQTT,
        options: {
          url: 'mqtt://localhost:1883',
        }
      },
    ]),
  ]
  ...
})
```

也可以使用其他创建客户端的选项（`ClientProxyFactory`或`@Client()`）。你可以在<a href="https://docs.nestjs.com/microservices/basics#client">此处</a>了解它们。

#### 上下文

在更复杂的场景中，你可能需要访问有关传入请求的附加信息。使用MQTT传输器时，可以访问`MqttContext`对象。

```typescript
@@filename()
@MessagePattern('notifications')
getNotifications(@Payload() data: number[], @Ctx() context: MqttContext) {
  console.log(`Topic: ${context.getTopic()}`);
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('notifications')
getNotifications(data, context) {
  console.log(`Topic: ${context.getTopic()}`);
}
```

> info **提示** `@Payload()`、`@Ctx()`和`MqttContext`从`@nestjs/microservices`包导入。

要访问原始的mqtt[数据包](https://github.com/mqttjs/mqtt-packet)，请使用`MqttContext`对象的`getPacket()`方法，如下所示：

```typescript
@@filename()
@MessagePattern('notifications')
getNotifications(@Payload() data: number[], @Ctx() context: MqttContext) {
  console.log(context.getPacket());
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('notifications')
getNotifications(data, context) {
  console.log(context.getPacket());
}
```

#### 通配符

订阅可以是明确的主题，也可以包含通配符。有两种通配符可用：`+`和`#`。`+`是单级通配符，而`#`是多级通配符，可覆盖多个主题级别。

```typescript
@@filename()
@MessagePattern('sensors/+/temperature/+')
getTemperature(@Ctx() context: MqttContext) {
  console.log(`Topic: ${context.getTopic()}`);
}
@@switch
@Bind(Ctx())
@MessagePattern('sensors/+/temperature/+')
getTemperature(context) {
  console.log(`Topic: ${context.getTopic()}`);
}
```

#### 服务质量（QoS）

使用`@MessagePattern`或`@EventPattern`装饰器创建的任何订阅都将以QoS 0进行订阅。如果需要更高的QoS，可以在建立连接时使用`subscribeOptions`块全局设置，如下所示：

```typescript
@@filename(main)
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.MQTT,
  options: {
    url: 'mqtt://localhost:1883',
    subscribeOptions: {
      qos: 2
    },
  },
});
@@switch
const app = await NestFactory.createMicroservice(AppModule, {
  transport: Transport.MQTT,
  options: {
    url: 'mqtt://localhost:1883',
    subscribeOptions: {
      qos: 2
    },
  },
});
```

如果需要特定于主题的QoS，请考虑创建[自定义传输器](https://docs.nestjs.com/microservices/custom-transport)。

#### 记录构建器

要配置消息选项（调整QoS级别、设置Retain或DUP标志，或向有效负载添加附加属性），可以使用`MqttRecordBuilder`类。例如，要将`QoS`设置为`2`，请使用`setQoS`方法，如下所示：

```typescript
const userProperties = { 'x-version': '1.0.0' };
const record = new MqttRecordBuilder(':cat:')
  .setProperties({ userProperties })
  .setQoS(1)
  .build();
client.send('replace-emoji', record).subscribe(...);
```

> info **提示** `MqttRecordBuilder`类从`@nestjs/microservices`包导出。

你也可以在服务器端通过访问`MqttContext`读取这些选项。

```typescript
@@filename()
@MessagePattern('replace-emoji')
replaceEmoji(@Payload() data: string, @Ctx() context: MqttContext): string {
  const { properties: { userProperties } } = context.getPacket();
  return userProperties['x-version'] === '1.0.0' ? '🐱' : '🐈';
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('replace-emoji')
replaceEmoji(data, context) {
  const { properties: { userProperties } } = context.getPacket();
  return userProperties['x-version'] === '1.0.0' ? '🐱' : '🐈';
}
```

在某些情况下，你可能希望为多个请求配置用户属性，可以将这些选项传递给`ClientProxyFactory`。

```typescript
import { Module } from '@nestjs/common';
import { ClientProxyFactory, Transport } from '@nestjs/microservices';

@Module({
  providers: [
    {
      provide: 'API_v1',
      useFactory: () =>
        ClientProxyFactory.create({
          transport: Transport.MQTT,
          options: {
            url: 'mqtt://localhost:1833',
            userProperties: { 'x-version': '1.0.0' },
          },
        }),
    },
  ],
})
export class ApiModule {}
```

#### 实例状态更新

要获取有关连接和底层驱动实例状态的实时更新，可以订阅`status`流。该流提供特定于所选驱动的状态更新。对于MQTT驱动，`status`流会发出`connected`、`disconnected`、`reconnecting`和`closed`事件。

```typescript
this.client.status.subscribe((status: MqttStatus) => {
  console.log(status);
});
```

> info **提示** `MqttStatus`类型从`@nestjs/microservices`包导入。

同样，可以订阅服务器的`status`流以接收有关服务器状态的通知。

```typescript
const server = app.connectMicroservice<MicroserviceOptions>(...);
server.status.subscribe((status: MqttStatus) => {
  console.log(status);
});
```

#### 监听MQTT事件

在某些情况下，你可能希望监听微服务发出的内部事件。例如，你可以监听`error`事件，以便在发生错误时触发额外的操作。要做到这一点，请使用`on()`方法，如下所示：

```typescript
this.client.on('error', (err) => {
  console.error(err);
});
```

同样，可以监听服务器的内部事件：

```typescript
server.on<MqttEvents>('error', (err) => {
  console.error(err);
});
```

> info **提示** `MqttEvents`类型从`@nestjs/microservices`包导入。

#### 底层驱动访问

对于更高级的用例，你可能需要访问底层驱动实例。这在诸如手动关闭连接或使用驱动特定方法等场景中可能很有用。但是，请记住，在大多数情况下，你**不需要**直接访问驱动。

要访问底层驱动实例，可以使用`unwrap()`方法，该方法返回底层驱动实例。泛型类型参数应指定你期望的驱动实例类型。

```typescript
const mqttClient = this.client.unwrap<import('mqtt').MqttClient>();
```

同样，可以访问服务器的底层驱动实例：

```typescript
const mqttClient = server.unwrap<import('mqtt').MqttClient>();
```