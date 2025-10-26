### gRPC

[gRPC](https://github.com/grpc/grpc-node) 是一个现代的、开源的、高性能的RPC框架，可在任何环境中运行。它能高效地连接数据中心内外的服务，并提供可插拔的负载均衡、追踪、健康检查和认证支持。

与许多RPC系统一样，gRPC基于这样一种概念：根据可远程调用的函数（方法）来定义服务。对于每个方法，你需要定义参数和返回类型。服务、参数和返回类型在`.proto`文件中使用谷歌的开源语言中立的<a href="https://protobuf.dev">协议缓冲区</a>机制进行定义。

借助gRPC传输器，Nest使用`.proto`文件动态绑定客户端和服务器，以便轻松实现远程过程调用，自动序列化和反序列化结构化数据。

#### 安装

要开始构建基于gRPC的微服务，首先安装所需的包：

```bash
$ npm i --save @grpc/grpc-js @grpc/proto-loader
```

#### 概述

与其他Nest微服务传输层实现一样，你可以通过传递给`createMicroservice()`方法的选项对象的`transport`属性来选择gRPC传输器机制。在下面的示例中，我们将设置一个英雄服务。`options`属性提供了该服务的元数据；其属性在<a href="microservices/grpc#options">下方</a>进行描述。

```typescript
@@filename(main)
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.GRPC,
  options: {
    package: 'hero',
    protoPath: join(__dirname, 'hero/hero.proto'),
  },
});
@@switch
const app = await NestFactory.createMicroservice(AppModule, {
  transport: Transport.GRPC,
  options: {
    package: 'hero',
    protoPath: join(__dirname, 'hero/hero.proto'),
  },
});
```

> 信息 **提示** `join()`函数从`path`包导入；`Transport`枚举从`@nestjs/microservices`包导入。

在`nest-cli.json`文件中，我们添加`assets`属性，允许分发非TypeScript文件，以及`watchAssets`——用于开启对所有非TypeScript资源的监视。在我们的案例中，我们希望`.proto`文件自动复制到`dist`文件夹。

```json
{
  "compilerOptions": {
    "assets": ["**/*.proto"],
    "watchAssets": true
  }
}
```

#### 选项

<strong>gRPC</strong>传输器选项对象公开了如下所述的属性。

<table>
  <tr>
    <td><code>package</code></td>
    <td>Protobuf包名（与<code>.proto</code>文件中的<code>package</code>设置匹配）。必需</td>
  </tr>
  <tr>
    <td><code>protoPath</code></td>
    <td>
      <code>.proto</code>文件的绝对（或相对于根目录的）路径。必需
    </td>
  </tr>
  <tr>
    <td><code>url</code></td>
    <td>连接URL。格式为<code>IP地址/域名:端口</code>的字符串（例如，Docker服务器使用<code>'0.0.0.0:50051'</code>），定义传输器建立连接的地址/端口。可选。默认为<code>'localhost:5000'</code></td>
  </tr>
  <tr>
    <td><code>protoLoader</code></td>
    <td>用于加载<code>.proto</code>文件的工具的NPM包名。可选。默认为<code>'@grpc/proto-loader'</code></td>
  </tr>
  <tr>
    <td><code>loader</code></td>
    <td>
      <code>@grpc/proto-loader</code>选项。这些选项提供对<code>.proto</code>文件行为的详细控制。可选。更多详情参见
      <a
        href="https://github.com/grpc/grpc-node/blob/master/packages/proto-loader/README.md"
        rel="nofollow"
        target="_blank"
        >此处</a
      >
    </td>
  </tr>
  <tr>
    <td><code>credentials</code></td>
    <td>
      服务器凭据。可选。<a
        href="https://grpc.io/grpc/node/grpc.ServerCredentials.html"
        rel="nofollow"
        target="_blank"
        >在此处了解更多</a
      >
    </td>
  </tr>
</table>

#### 示例gRPC服务

让我们定义名为`HeroesService`的示例gRPC服务。在上面的`options`对象中，`protoPath`属性设置了指向`.proto`定义文件`hero.proto`的路径。`hero.proto`文件使用<a href="https://developers.google.com/protocol-buffers">协议缓冲区</a>进行结构化。如下所示：

```typescript
// hero/hero.proto
syntax = "proto3";

package hero;

service HeroesService {
  rpc FindOne (HeroById) returns (Hero) {}
}

message HeroById {
  int32 id = 1;
}

message Hero {
  int32 id = 1;
  string name = 2;
}
```

我们的`HeroesService`公开了`FindOne()`方法。该方法期望一个`HeroById`类型的输入参数，并返回一个`Hero`消息（协议缓冲区使用`message`元素来定义参数类型和返回类型）。

接下来，我们需要实现该服务。要定义一个满足此定义的处理器，我们在控制器中使用`@GrpcMethod()`装饰器，如下所示。该装饰器提供了将方法声明为gRPC服务方法所需的元数据。

> 信息 **提示** 之前微服务章节中介绍的`@MessagePattern()`装饰器（<a href="microservices/basics#request-response">了解更多</a>）不用于基于gRPC的微服务。对于基于gRPC的微服务，`@GrpcMethod()`装饰器实际上取代了它的位置。

```typescript
@@filename(heroes.controller)
@Controller()
export class HeroesController {
  @GrpcMethod('HeroesService', 'FindOne')
  findOne(data: HeroById, metadata: Metadata, call: ServerUnaryCall<any, any>): Hero {
    const items = [
      { id: 1, name: 'John' },
      { id: 2, name: 'Doe' },
    ];
    return items.find(({ id }) => id === data.id);
  }
}
@@switch
@Controller()
export class HeroesController {
  @GrpcMethod('HeroesService', 'FindOne')
  findOne(data, metadata, call) {
    const items = [
      { id: 1, name: 'John' },
      { id: 2, name: 'Doe' },
    ];
    return items.find(({ id }) => id === data.id);
  }
}
```

> 信息 **提示** `@GrpcMethod()`装饰器从`@nestjs/microservices`包导入，而`Metadata`和`ServerUnaryCall`从`grpc`包导入。

上面显示的装饰器接受两个参数。第一个是服务名称（例如`'HeroesService'`），对应于`hero.proto`中的`HeroesService`服务定义。第二个（字符串`'FindOne'`）对应于`hero.proto`文件中`HeroesService`内定义的`FindOne()`rpc方法。

`findOne()`处理器方法接受三个参数：从调用者传递来的`data`、存储gRPC请求元数据的`metadata`，以及用于获取`GrpcCall`对象属性（如用于向客户端发送元数据的`sendMetadata`）的`call`。

`@GrpcMethod()`装饰器的两个参数都是可选的。如果不传递第二个参数（例如`'FindOne'`），Nest会自动将`.proto`文件中的rpc方法与处理器关联，关联方式是将处理器名称转换为大驼峰式（例如，`findOne`处理器与`FindOne`rpc调用定义关联）。如下所示：

```typescript
@@filename(heroes.controller)
@Controller()
export class HeroesController {
  @GrpcMethod('HeroesService')
  findOne(data: HeroById, metadata: Metadata, call: ServerUnaryCall<any, any>): Hero {
    const items = [
      { id: 1, name: 'John' },
      { id: 2, name: 'Doe' },
    ];
    return items.find(({ id }) => id === data.id);
  }
}
@@switch
@Controller()
export class HeroesController {
  @GrpcMethod('HeroesService')
  findOne(data, metadata, call) {
    const items = [
      { id: 1, name: 'John' },
      { id: 2, name: 'Doe' },
    ];
    return items.find(({ id }) => id === data.id);
  }
}
```

你也可以省略`@GrpcMethod()`的第一个参数。在这种情况下，Nest会根据定义处理器的**类**名，自动将处理器与proto定义文件中的服务定义关联。例如，在下面的代码中，`HeroesService`类基于名称`'HeroesService'`的匹配，将其处理器方法与`hero.proto`文件中的`HeroesService`服务定义关联。

```typescript
@@filename(heroes.controller)
@Controller()
export class HeroesService {
  @GrpcMethod()
  findOne(data: HeroById, metadata: Metadata, call: ServerUnaryCall<any, any>): Hero {
    const items = [
      { id: 1, name: 'John' },
      { id: 2, name: 'Doe' },
    ];
    return items.find(({ id }) => id === data.id);
  }
}
@@switch
@Controller()
export class HeroesService {
  @GrpcMethod()
  findOne(data, metadata, call) {
    const items = [
      { id: 1, name: 'John' },
      { id: 2, name: 'Doe' },
    ];
    return items.find(({ id }) => id === data.id);
  }
}
```

#### 客户端

Nest应用程序可以作为gRPC客户端，消费`.proto`文件中定义的服务。你可以通过`ClientGrpc`对象访问远程服务。你可以通过多种方式获取`ClientGrpc`对象。

首选方法是导入`ClientsModule`。使用`register()`方法将`.proto`文件中定义的服务包绑定到注入令牌，并配置服务。`name`属性是注入令牌。对于gRPC服务，使用`transport: Transport.GRPC`。`options`属性是一个对象，具有上面<a href="microservices/grpc#options">描述的</a>相同属性。

```typescript
imports: [
  ClientsModule.register([
    {
      name: 'HERO_PACKAGE',
      transport: Transport.GRPC,
      options: {
        package: 'hero',
        protoPath: join(__dirname, 'hero/hero.proto'),
      },
    },
  ]),
];
```

> 信息 **提示** `register()`方法接受一个对象数组。通过提供逗号分隔的注册对象列表来注册多个包。

注册后，我们可以使用`@Inject()`注入配置好的`ClientGrpc`对象。然后，我们使用`ClientGrpc`对象的`getService()`方法检索服务实例，如下所示：

```typescript
@Injectable()
export class AppService implements OnModuleInit {
  private heroesService: HeroesService;

  constructor(@Inject('HERO_PACKAGE') private client: ClientGrpc) {}

  onModuleInit() {
    this.heroesService = this.client.getService<HeroesService>('HeroesService');
  }

  getHero(): Observable<string> {
    return this.heroesService.findOne({ id: 1 });
  }
}
```

> 错误 **警告** 除非在proto加载器配置（微服务传输器配置中的`options.loader.keepcase`）中将`keepCase`选项设置为`true`，否则gRPC客户端不会发送名称中包含下划线`_`的字段。

注意，与其他微服务传输方法中使用的技术相比有一个小差异。我们使用`ClientGrpc`类而不是`ClientProxy`类，`ClientGrpc`类提供`getService()`方法。`getService()`泛型方法接受服务名称作为参数，并返回其实例（如果可用）。

或者，你可以使用`@Client()`装饰器实例化`ClientGrpc`对象，如下所示：

```typescript
@Injectable()
export class AppService implements OnModuleInit {
  @Client({
    transport: Transport.GRPC,
    options: {
      package: 'hero',
      protoPath: join(__dirname, 'hero/hero.proto'),
    },
  })
  client: ClientGrpc;

  private heroesService: HeroesService;

  onModuleInit() {
    this.heroesService = this.client.getService<HeroesService>('HeroesService');
  }

  getHero(): Observable<string> {
    return this.heroesService.findOne({ id: 1 });
  }
}
```

最后，对于更复杂的场景，我们可以使用`ClientProxyFactory`类注入动态配置的客户端，如<a href="/microservices/basics#client">此处</a>所述。

无论哪种情况，我们最终都会获得`HeroesService`代理对象的引用，该对象公开与`.proto`文件中定义的相同的方法集。现在，当我们访问这个代理对象（即`heroesService`）时，gRPC系统会自动序列化请求，将其转发到远程系统，返回响应，并反序列化响应。因为gRPC为我们屏蔽了这些网络通信细节，所以`heroesService`看起来和作用起来就像一个本地提供者。

注意，所有服务方法都是**小驼峰式**的（为了遵循语言的自然约定）。例如，虽然我们的`.proto`文件`HeroesService`定义包含`FindOne()`函数，但`heroesService`实例将提供`findOne()`方法。

```typescript
interface HeroesService {
  findOne(data: { id: number }): Observable<any>;
}
```

消息处理器也能够返回`Observable`，在这种情况下，结果值将被发射，直到流完成。

```typescript
@@filename(heroes.controller)
@Get()
call(): Observable<any> {
  return this.heroesService.findOne({ id: 1 });
}
@@switch
@Get()
call() {
  return this.heroesService.findOne({ id: 1 });
}
```

要发送gRPC元数据（连同请求一起），你可以传递第二个参数，如下所示：

```typescript
call(): Observable<any> {
  const metadata = new Metadata();
  metadata.add('Set-Cookie', 'yummy_cookie=choco');

  return this.heroesService.findOne({ id: 1 }, metadata);
}
```

> 信息 **提示** `Metadata`类从`grpc`包导入。

请注意，这需要更新我们前面定义的`HeroesService`接口。

#### 示例

可在此处获取工作示例[here](https://github.com/nestjs/nest/tree/master/sample/04-grpc)。

#### gRPC反射

[gRPC服务器反射规范](https://grpc.io/docs/guides/reflection/#overview)是一个标准，允许gRPC客户端请求服务器公开的API的详细信息，类似于为REST API公开OpenAPI文档。这可以使使用grpc-ui或postman等开发人员调试工具变得更加容易。

要为服务器添加gRPC反射支持，首先安装所需的实现包：

```bash
$ npm i --save @grpc/reflection
```

然后，可以使用gRPC服务器选项中的`onLoadPackageDefinition`钩子将其挂钩到gRPC服务器，如下所示：

```typescript
@@filename(main)
import { ReflectionService } from '@grpc/reflection';

const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  options: {
    onLoadPackageDefinition: (pkg, server) => {
      new ReflectionService(pkg).addToServer(server);
    },
  },
});
```

现在，你的服务器将使用反射规范响应请求API详情的消息。

#### gRPC流

gRPC本身支持长期的实时连接，通常称为`流（streams）`。流在聊天、观测或块数据传输等情况下非常有用。在官方文档中可找到更多详情[here](https://grpc.io/docs/guides/concepts/)。

Nest支持两种可能的GRPC流处理器方式：

- RxJS `Subject` + `Observable`处理器：可用于在Controller方法内部直接编写响应，或传递给`Subject`/`Observable`消费者
- 纯GRPC调用流处理器：可用于传递给某个执行器，该执行器将处理Node标准`Duplex`流处理器的其余分发

<app-banner-enterprise></app-banner-enterprise>

#### 流示例

让我们定义一个名为`HelloService`的新示例gRPC服务。`hello.proto`文件使用<a href="https://developers.google.com/protocol-buffers">协议缓冲区</a>进行结构化。如下所示：

```typescript
// hello/hello.proto
syntax = "proto3";

package hello;

service HelloService {
  rpc BidiHello(stream HelloRequest) returns (stream HelloResponse);
  rpc LotsOfGreetings(stream HelloRequest) returns (HelloResponse);
}

message HelloRequest {
  string greeting = 1;
}

message HelloResponse {
  string reply = 1;
}
```

> 信息 **提示** `LotsOfGreetings`方法可以简单地使用`@GrpcMethod`装饰器实现（如上面的示例所示），因为返回的流可以发射多个值。

基于此`.proto`文件，让我们定义`HelloService`接口：

```typescript
interface HelloService {
  bidiHello(upstream: Observable<HelloRequest>): Observable<HelloResponse>;
  lotsOfGreetings(
    upstream: Observable<HelloRequest>,
  ): Observable<HelloResponse>;
}

interface HelloRequest {
  greeting: string;
}

interface HelloResponse {
  reply: string;
}
```

> 信息 **提示** 可以通过[ts-proto](https://github.com/stephenh/ts-proto)包自动生成proto接口，更多信息参见[here](https://github.com/stephenh/ts-proto/blob/main/NESTJS.markdown)。

#### Subject策略

`@GrpcStreamMethod()`装饰器将函数参数作为RxJS `Observable`提供。因此，我们可以接收和处理多个消息。

```typescript
@GrpcStreamMethod()
bidiHello(messages: Observable<any>, metadata: Metadata, call: ServerDuplexStream<any, any>): Observable<any> {
  const subject = new Subject();

  const onNext = message => {
    console.log(message);
    subject.next({
      reply: 'Hello, world!'
    });
  };
  const onComplete = () => subject.complete();
  messages.subscribe({
    next: onNext,
    complete: onComplete,
  });


  return subject.asObservable();
}
```

> 警告 **警告** 为了支持与`@GrpcStreamMethod()`装饰器的全双工交互，控制器方法必须返回RxJS `Observable`。

> 信息 **提示** `Metadata`和`ServerUnaryCall`类/接口从`grpc`包导入。

根据服务定义（在`.proto`文件中），`BidiHello`方法应该将请求流式传输到服务。要从客户端向流发送多个异步消息，我们利用RxJS `ReplaySubject`类。

```typescript
const helloService = this.client.getService<HelloService>('HelloService');
const helloRequest$ = new ReplaySubject<HelloRequest>();

helloRequest$.next({ greeting: 'Hello (1)!' });
helloRequest$.next({ greeting: 'Hello (2)!' });
helloRequest$.complete();

return helloService.bidiHello(helloRequest$);
```

在上面的示例中，我们向流写入了两条消息（`next()`调用），并通知服务我们已完成数据发送（`complete()`调用）。

#### 调用流处理器

当方法返回值定义为`stream`时，`@GrpcStreamCall()`装饰器将函数参数作为`grpc.ServerDuplexStream`提供，该流支持`.on('data', callback)`、`.write(message)`或`.cancel()`等标准方法。关于可用方法的完整文档可在[here](https://grpc.github.io/grpc/node/grpc-ClientDuplexStream.html)找到。

或者，当方法返回值不是`stream`时，`@GrpcStreamCall()`装饰器提供两个函数参数，分别是`grpc.ServerReadableStream`（更多信息参见[here](https://grpc.github.io/grpc/node/grpc-ServerReadableStream.html)）和`callback`。

让我们开始实现应该支持全双工交互的`BidiHello`。

```typescript
@GrpcStreamCall()
bidiHello(requestStream: any) {
  requestStream.on('data', message => {
    console.log(message);
    requestStream.write({
      reply: 'Hello, world!'
    });
  });
}
```

> 信息 **提示** 此装饰器不需要提供任何特定的返回参数。预计流将以与其他标准流类型类似的方式处理。

在上面的示例中，我们使用`write()`方法将对象写入响应流。作为第二个参数传递给`.on()`方法的回调将在我们的服务每次接收新的数据块时被调用。

让我们实现`LotsOfGreetings`方法。

```typescript
@GrpcStreamCall()
lotsOfGreetings(requestStream: any, callback: (err: unknown, value: HelloResponse) => void) {
  requestStream.on('data', message => {
    console.log(message);
  });
  requestStream.on('end', () => callback(null, { reply: 'Hello, world!' }));
}
```

在这里，我们使用`callback`函数在`requestStream`处理完成后发送响应。

#### gRPC元数据

元数据是关于特定RPC调用的信息，以键值对列表的形式存在，其中键是字符串，值通常是字符串，但也可以是二进制数据。元数据对gRPC本身是不透明的——它允许客户端向服务器提供与调用相关的信息，反之亦然。元数据可能包括认证令牌、请求标识符和用于监控目的的标签，以及数据集记录数等数据信息。

要在`@GrpcMethod()`处理器中读取元数据，请使用第二个参数（metadata），其类型为`Metadata`（从`grpc`包导入）。

要从处理器发送回元数据，请使用`ServerUnaryCall#sendMetadata()`方法（第三个处理器参数）。

```typescript
@@filename(heroes.controller)
@Controller()
export class HeroesService {
  @GrpcMethod()
  findOne(data: HeroById, metadata: Metadata, call: ServerUnaryCall<any, any>): Hero {
    const serverMetadata = new Metadata();
    const items = [
      { id: 1, name: 'John' },
      { id: 2, name: 'Doe' },
    ];

    serverMetadata.add('Set-Cookie', 'yummy_cookie=choco');
    call.sendMetadata(serverMetadata);

    return items.find(({ id }) => id === data.id);
  }
}
@@switch
@Controller()
export class HeroesService {
  @GrpcMethod()
  findOne(data, metadata, call) {
    const serverMetadata = new Metadata();
    const items = [
      { id: 1, name: 'John' },
      { id: 2, name: 'Doe' },
    ];

    serverMetadata.add('Set-Cookie', 'yummy_cookie=choco');
    call.sendMetadata(serverMetadata);

    return items.find(({ id }) => id === data.id);
  }
}
```

同样，要在使用`@GrpcStreamMethod()`处理器（[subject策略](microservices/grpc#subject-strategy)）注释的处理器中读取元数据，请使用第二个参数（metadata），其类型为`Metadata`（从`grpc`包导入）。

要从处理器发送回元数据，请使用`ServerDuplexStream#sendMetadata()`方法（第三个处理器参数）。

要从[调用流处理器](microservices/grpc#call-stream-handler)（使用`@GrpcStreamCall()`装饰器注释的处理器）中读取元数据，请在`requestStream`引用上监听`metadata`事件，如下所示：

```typescript
requestStream.on('metadata', (metadata: Metadata) => {
  const meta = metadata.get('X-Meta');
});
```
