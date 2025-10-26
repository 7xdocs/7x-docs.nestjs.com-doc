### 适配器

WebSockets 模块是与平台无关的，因此，您可以通过利用 `WebSocketAdapter` 接口来引入自己的库（甚至是原生实现）。该接口强制实现下表中描述的少数方法：

<table>
  <tr>
    <td><code>create</code></td>
    <td>基于传入的参数创建一个 socket 实例</td>
  </tr>
  <tr>
    <td><code>bindClientConnect</code></td>
    <td>绑定客户端连接事件</td>
  </tr>
  <tr>
    <td><code>bindClientDisconnect</code></td>
    <td>绑定客户端断开连接事件（可选*）</td>
  </tr>
  <tr>
    <td><code>bindMessageHandlers</code></td>
    <td>将传入消息绑定到相应的消息处理器</td>
  </tr>
  <tr>
    <td><code>close</code></td>
    <td>终止服务器实例</td>
  </tr>
</table>

#### 扩展 socket.io

[socket.io](https://github.com/socketio/socket.io) 包被包装在 `IoAdapter` 类中。如果您想增强此适配器的基本功能该怎么办？例如，您的技术需求要求能够跨 Web 服务的多个负载均衡实例广播事件。为此，您可以扩展 `IoAdapter` 并重写单个方法，该方法的职责是实例化新的 socket.io 服务器。但首先，让我们安装所需的包。

> warning **警告** 要将 socket.io 用于多个负载均衡实例，您必须通过在客户端的 socket.io 配置中设置 `transports: ['websocket']` 来禁用轮询，或者必须在负载均衡器中启用基于 cookie 的路由。仅使用 Redis 是不够的。更多信息请参阅[此处](https://socket.io/docs/v4/using-multiple-nodes/#enabling-sticky-session)。

```bash
$ npm i --save redis socket.io @socket.io/redis-adapter
```

一旦包安装完成，我们就可以创建一个 `RedisIoAdapter` 类。

```typescript
import { IoAdapter } from '@nestjs/platform-socket.io';
import { ServerOptions } from 'socket.io';
import { createAdapter } from '@socket.io/redis-adapter';
import { createClient } from 'redis';

export class RedisIoAdapter extends IoAdapter {
  private adapterConstructor: ReturnType<typeof createAdapter>;

  async connectToRedis(): Promise<void> {
    const pubClient = createClient({ url: `redis://localhost:6379` });
    const subClient = pubClient.duplicate();

    await Promise.all([pubClient.connect(), subClient.connect()]);

    this.adapterConstructor = createAdapter(pubClient, subClient);
  }

  createIOServer(port: number, options?: ServerOptions): any {
    const server = super.createIOServer(port, options);
    server.adapter(this.adapterConstructor);
    return server;
  }
}
```

之后，只需切换到新创建的 Redis 适配器。

```typescript
const app = await NestFactory.create(AppModule);
const redisIoAdapter = new RedisIoAdapter(app);
await redisIoAdapter.connectToRedis();

app.useWebSocketAdapter(redisIoAdapter);
```

#### Ws 库

另一个可用的适配器是 `WsAdapter`，它充当框架与集成的极速且经过全面测试的 [ws](https://github.com/websockets/ws) 库之间的代理。该适配器与原生浏览器 WebSockets 完全兼容，并且比 socket.io 包快得多。不幸的是，它开箱即用的功能要少得多。但在某些情况下，您可能并不需要它们。

> info **提示** `ws` 库不支持命名空间（由 `socket.io` 推广的通信通道）。但是，为了模拟此功能，您可以在不同的路径上挂载多个 `ws` 服务器（例如：`@WebSocketGateway({{ '{' }} path: '/users' {{ '}' }})`）。

为了使用 `ws`，我们首先必须安装所需的包：

```bash
$ npm i --save @nestjs/platform-ws
```

一旦包安装完成，我们就可以切换适配器：

```typescript
const app = await NestFactory.create(AppModule);
app.useWebSocketAdapter(new WsAdapter(app));
```

> info **提示** `WsAdapter` 是从 `@nestjs/platform-ws` 导入的。

`wsAdapter` 被设计为处理 `{{ '{' }} event: string, data: any {{ '}' }}` 格式的消息。如果您需要以不同格式接收和处理消息，您需要配置一个消息解析器将其转换为所需的格式。

```typescript
const wsAdapter = new WsAdapter(app, {
  // 处理 [event, data] 格式的消息
  messageParser: (data) => {
    const [event, payload] = JSON.parse(data.toString());
    return { event, data: payload };
  },
});
```

或者，您可以在适配器创建后使用 `setMessageParser` 方法配置消息解析器。

#### 高级（自定义适配器）

出于演示目的，我们将手动集成 [ws](https://github.com/websockets/ws) 库。如前所述，该库的适配器已经创建，并从 `@nestjs/platform-ws` 包中作为 `WsAdapter` 类暴露。以下是其简化实现的可能样子：

```typescript
@@filename(ws-adapter)
import * as WebSocket from 'ws';
import { WebSocketAdapter, INestApplicationContext } from '@nestjs/common';
import { MessageMappingProperties } from '@nestjs/websockets';
import { Observable, fromEvent, EMPTY } from 'rxjs';
import { mergeMap, filter } from 'rxjs/operators';

export class WsAdapter implements WebSocketAdapter {
  constructor(private app: INestApplicationContext) {}

  create(port: number, options: any = {}): any {
    return new WebSocket.Server({ port, ...options });
  }

  bindClientConnect(server, callback: Function) {
    server.on('connection', callback);
  }

  bindMessageHandlers(
    client: WebSocket,
    handlers: MessageMappingProperties[],
    process: (data: any) => Observable<any>,
  ) {
    fromEvent(client, 'message')
      .pipe(
        mergeMap(data => this.bindMessageHandler(data, handlers, process)),
        filter(result => result),
      )
      .subscribe(response => client.send(JSON.stringify(response)));
  }

  bindMessageHandler(
    buffer,
    handlers: MessageMappingProperties[],
    process: (data: any) => Observable<any>,
  ): Observable<any> {
    const message = JSON.parse(buffer.data);
    const messageHandler = handlers.find(
      handler => handler.message === message.event,
    );
    if (!messageHandler) {
      return EMPTY;
    }
    return process(messageHandler.callback(message.data));
  }

  close(server) {
    server.close();
  }
}
```

> info **提示** 当您想要利用 [ws](https://github.com/websockets/ws) 库时，请使用内置的 `WsAdapter` 而不是创建自己的。

然后，我们可以使用 `useWebSocketAdapter()` 方法设置自定义适配器：

```typescript
@@filename(main)
const app = await NestFactory.create(AppModule);
app.useWebSocketAdapter(new WsAdapter(app));
```

#### 示例

一个使用 `WsAdapter` 的工作示例可在[此处](https://github.com/nestjs/nest/tree/master/sample/16-gateways-ws)找到。