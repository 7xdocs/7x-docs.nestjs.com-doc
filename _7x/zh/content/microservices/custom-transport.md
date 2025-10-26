### 自定义传输器

Nest 提供了多种开箱即用的**传输器**，以及一个允许开发者构建新的自定义传输策略的 API。传输器使你能够通过可插拔的通信层和非常简单的应用级消息协议在网络上连接组件（阅读完整[文章](https://dev.to/nestjs/integrate-nestjs-with-external-services-using-microservice-transporters-part-1-p3)）。

> 信息 **提示** 使用 Nest 构建微服务并不一定意味着你必须使用 `@nestjs/microservices` 包。例如，如果你想与外部服务（假设是用不同语言编写的其他微服务）通信，你可能不需要 `@nestjs/microservice` 库提供的所有功能。
> 实际上，如果你不需要那些允许你声明性定义订阅者的装饰器（`@EventPattern` 或 `@MessagePattern`），运行一个[独立应用](/application-context)并手动维护连接/订阅频道对于大多数用例来说已经足够，并且会为你提供更大的灵活性。

借助自定义传输器，你可以集成任何消息系统/协议（包括 Google Cloud Pub/Sub、Amazon Kinesis 等），或者扩展现有系统，在其之上添加额外功能（例如，MQTT 的 [QoS](https://github.com/mqttjs/MQTT.js/blob/master/README.md#qos)）。

> 信息 **提示** 为了更好地理解 Nest 微服务的工作原理以及如何扩展现有传输器的功能，我们建议阅读 [NestJS Microservices in Action](https://dev.to/johnbiundo/series/4724) 和 [Advanced NestJS Microservices](https://dev.to/nestjs/part-1-introduction-and-setup-1a2l) 文章系列。

#### 创建策略

首先，让我们定义一个表示自定义传输器的类。

```typescript
import { CustomTransportStrategy, Server } from '@nestjs/microservices';

class GoogleCloudPubSubServer
  extends Server
  implements CustomTransportStrategy {
  /**
   * This method is triggered when you run "app.listen()".
   */
  listen(callback: () => void) {
    callback();
  }

  /**
   * This method is triggered on application shutdown.
   */
  close() {}
}
```

> 警告 **注意** 请注意，在本章中我们不会实现一个功能完备的 Google Cloud Pub/Sub 服务器，因为这需要深入研究传输器特定的技术细节。

在上面的示例中，我们声明了 `GoogleCloudPubSubServer` 类，并提供了 `CustomTransportStrategy` 接口所要求的 `listen()` 和 `close()` 方法。
此外，我们的类继承了从 `@nestjs/microservices` 包导入的 `Server` 类，该类提供了一些有用的方法，例如 Nest 运行时用于注册消息处理程序的方法。或者，如果你想扩展现有传输策略的功能，你可以扩展相应的服务器类，例如 `ServerRedis`。
按照惯例，我们在类名后添加了“Server”后缀，因为它将负责订阅消息/事件（并在必要时响应它们）。

完成这些后，我们现在可以使用自定义策略代替内置传输器，如下所示：

```typescript
const app = await NestFactory.createMicroservice<MicroserviceOptions>(
  AppModule,
  {
    strategy: new GoogleCloudPubSubServer(),
  },
);
```

基本上，我们没有传递带有 `transport` 和 `options` 属性的常规传输器选项对象，而是传递了一个单一属性 `strategy`，其值是我们自定义传输器类的实例。

回到我们的 `GoogleCloudPubSubServer` 类，在实际应用中，我们会在 `listen()` 方法中建立与消息代理/外部服务的连接并注册订阅者/监听特定频道（然后在 `close()` 拆卸方法中移除订阅并关闭连接），
但由于这需要很好地理解 Nest 微服务之间的通信方式，我们建议阅读这篇[文章系列](https://dev.to/nestjs/part-1-introduction-and-setup-1a2l)。
在本章中，我们将重点关注 `Server` 类提供的功能以及如何利用它们构建自定义策略。

例如，假设在我们的应用程序中的某个地方，定义了以下消息处理程序：

```typescript
@MessagePattern('echo')
echo(@Payload() data: object) {
  return data;
}
```

这个消息处理程序将由 Nest 运行时自动注册。通过 `Server` 类，你可以查看已注册的消息模式，还可以访问并执行分配给它们的实际方法。
为了测试这一点，让我们在调用 `callback` 函数之前，在 `listen()` 方法中添加一个简单的 `console.log`：

```typescript
listen(callback: () => void) {
  console.log(this.messageHandlers);
  callback();
}
```

应用程序重启后，你将在终端中看到以下日志：

```typescript
Map { 'echo' => [AsyncFunction] { isEventHandler: false } }
```

> 信息 **提示** 如果我们使用 `@EventPattern` 装饰器，你将看到相同的输出，但 `isEventHandler` 属性将设置为 `true`。

如你所见，`messageHandlers` 属性是所有消息（和事件）处理程序的 `Map` 集合，其中模式用作键。
现在，你可以使用一个键（例如，`"echo"`）来获取消息处理程序的引用：

```typescript
async listen(callback: () => void) {
  const echoHandler = this.messageHandlers.get('echo');
  console.log(await echoHandler('Hello world!'));
  callback();
}
```

当我们执行 `echoHandler` 并传递一个任意字符串作为参数（这里是 `"Hello world!"`）时，我们应该会在控制台中看到它：

```json
Hello world!
```

这意味着我们的方法处理程序已正确执行。

当将 `CustomTransportStrategy` 与[拦截器](/interceptors)一起使用时，处理程序会被包装到 RxJS 流中。这意味着你需要订阅它们才能执行流的底层逻辑（例如，在拦截器执行后继续进入控制器逻辑）。

下面是一个示例：

```typescript
async listen(callback: () => void) {
  const echoHandler = this.messageHandlers.get('echo');
  const streamOrResult = await echoHandler('Hello World');
  if (isObservable(streamOrResult)) {
    streamOrResult.subscribe();
  }
  callback();
}
```

#### 客户端代理

正如我们在第一部分中提到的，你不一定需要使用 `@nestjs/microservices` 包来创建微服务，但如果你决定使用它并且需要集成自定义策略，你也需要提供一个“客户端”类。

> 信息 **提示** 同样，实现一个与所有 `@nestjs/microservices` 功能（例如，流）兼容的功能完备的客户端类需要很好地理解框架使用的通信技术。要了解更多信息，请查看这篇[文章](https://dev.to/nestjs/part-4-basic-client-component-16f9)。

要与外部服务通信/发送和发布消息（或事件），你可以使用特定于库的 SDK 包，或者实现一个扩展 `ClientProxy` 的自定义客户端类，如下所示：

```typescript
import { ClientProxy, ReadPacket, WritePacket } from '@nestjs/microservices';

class GoogleCloudPubSubClient extends ClientProxy {
  async connect(): Promise<any> {}
  async close() {}
  async dispatchEvent(packet: ReadPacket<any>): Promise<any> {}
  publish(
    packet: ReadPacket<any>,
    callback: (packet: WritePacket<any>) => void,
  ): Function {}
}
```

> 警告 **注意** 请注意，在本章中我们不会实现一个功能完备的 Google Cloud Pub/Sub 客户端，因为这需要深入研究传输器特定的技术细节。

如你所见，`ClientProxy` 类要求我们提供几个用于建立和关闭连接以及发布消息（`publish`）和事件（`dispatchEvent`）的方法。
请注意，如果你不需要请求-响应通信风格支持，你可以让 `publish()` 方法为空。同样，如果你不需要支持基于事件的通信，可以跳过 `dispatchEvent()` 方法。

为了观察这些方法的执行内容和时间，让我们添加多个 `console.log` 调用，如下所示：

```typescript
class GoogleCloudPubSubClient extends ClientProxy {
  async connect(): Promise<any> {
    console.log('connect');
  }

  async close() {
    console.log('close');
  }

  async dispatchEvent(packet: ReadPacket<any>): Promise<any> {
    return console.log('event to dispatch: ', packet);
  }

  publish(
    packet: ReadPacket<any>,
    callback: (packet: WritePacket<any>) => void,
  ): Function {
    console.log('message:', packet);

    // In a real-world application, the "callback" function should be executed
    // with payload sent back from the responder. Here, we'll simply simulate (5 seconds delay)
    // that response came through by passing the same "data" as we've originally passed in.
    setTimeout(() => callback({ response: packet.data }), 5000);

    return () => console.log('teardown');
  }
}
```

完成这些后，让我们创建 `GoogleCloudPubSubClient` 类的实例并运行 `send()` 方法（你可能在前面的章节中见过），订阅返回的可观察流。

```typescript
const googlePubSubClient = new GoogleCloudPubSubClient();
googlePubSubClient
  .send('pattern', 'Hello world!')
  .subscribe((response) => console.log(response));
```

现在，你应该在终端中看到以下输出：

```typescript
connect
message: { pattern: 'pattern', data: 'Hello world!' }
Hello world! // <-- after 5 seconds
```

为了测试我们的“拆卸”方法（我们的 `publish()` 方法返回的方法）是否正确执行，让我们在流上应用一个超时操作符，将其设置为 2 秒，以确保它在我们的 `setTimeout` 调用 `callback` 函数之前抛出。

```typescript
const googlePubSubClient = new GoogleCloudPubSubClient();
googlePubSubClient
  .send('pattern', 'Hello world!')
  .pipe(timeout(2000))
  .subscribe(
    (response) => console.log(response),
    (error) => console.error(error.message),
  );
```

> 信息 **提示** `timeout` 操作符从 `rxjs/operators` 包导入。

应用 `timeout` 操作符后，你的终端输出应该如下所示：

```typescript
connect
message: { pattern: 'pattern', data: 'Hello world!' }
teardown // <-- teardown
Timeout has occurred
```

要分发事件（而不是发送消息），请使用 `emit()` 方法：

```typescript
googlePubSubClient.emit('event', 'Hello world!');
```

你应该在控制台中看到以下内容：

```typescript
connect
event to dispatch:  { pattern: 'event', data: 'Hello world!' }
```

#### 消息序列化

如果你需要在客户端围绕响应的序列化添加一些自定义逻辑，你可以使用一个扩展 `ClientProxy` 类或其任何子类的自定义类。要修改成功的请求，你可以重写 `serializeResponse` 方法，要修改通过此客户端的任何错误，你可以重写 `serializeError` 方法。要使用这个自定义类，你可以通过 `customClass` 属性将类本身传递给 `ClientsModule.register()` 方法。下面是一个自定义 `ClientProxy` 的示例，它将每个错误序列化为 `RpcException`。

```typescript
@@filename(error-handling.proxy)
import { ClientTcp, RpcException } from '@nestjs/microservices';

class ErrorHandlingProxy extends ClientTCP {
  serializeError(err: Error) {
    return new RpcException(err);
  }
}
```

然后在 `ClientsModule` 中这样使用它：

```typescript
@@filename(app.module)
@Module({
  imports: [
    ClientsModule.register([{
      name: 'CustomProxy',
      customClass: ErrorHandlingProxy,
    }]),
  ]
})
export class AppModule
```

> 信息 **提示** 传递给 `customClass` 的是类本身，而不是类的实例。Nest 会在底层为你创建实例，并将 `options` 属性中给出的任何选项传递给新的 `ClientProxy`。