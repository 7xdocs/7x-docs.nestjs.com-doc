### 事件

[Event Emitter](https://www.npmjs.com/package/@nestjs/event-emitter) 包 (`@nestjs/event-emitter`) 提供了一个简单的观察者模式实现，允许您订阅和监听应用程序中发生的各种事件。事件是解耦应用程序各个方面的好方法，因为单个事件可以有多个彼此不依赖的监听器。

`EventEmitterModule` 在内部使用 [eventemitter2](https://github.com/EventEmitter2/EventEmitter2) 包。

#### 入门

首先安装所需的包：

```shell
$ npm i --save @nestjs/event-emitter
```

安装完成后，将 `EventEmitterModule` 导入到根 `AppModule` 中，并运行 `forRoot()` 静态方法，如下所示：

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { EventEmitterModule } from '@nestjs/event-emitter';

@Module({
  imports: [
    EventEmitterModule.forRoot()
  ],
})
export class AppModule {}
```

`.forRoot()` 调用会初始化事件发射器，并注册应用程序中存在的任何声明性事件监听器。注册发生在 `onApplicationBootstrap` 生命周期钩子时，确保所有模块都已加载并声明了任何计划的任务。

要配置底层的 `EventEmitter` 实例，请将配置对象传递给 `.forRoot()` 方法，如下所示：

```typescript
EventEmitterModule.forRoot({
  // 将其设置为 `true` 以使用通配符
  wildcard: false,
  // 用于分割命名空间的分隔符
  delimiter: '.',
  // 如果希望发出 newListener 事件，则将其设置为 `true`
  newListener: false,
  // 如果希望发出 removeListener 事件，则将其设置为 `true`
  removeListener: false,
  // 可以分配给事件的最大监听器数量
  maxListeners: 10,
  // 当分配了超过最大数量的监听器时，在内存泄漏消息中显示事件名称
  verboseMemoryLeak: false,
  // 如果发出 error 事件且没有监听器，则禁用抛出 uncaughtException
  ignoreErrors: false,
});
```

#### 分发事件

要分发（即触发）事件，首先使用标准的构造函数注入方式注入 `EventEmitter2`：

```typescript
constructor(private eventEmitter: EventEmitter2) {}
```

> info **提示** 从 `@nestjs/event-emitter` 包中导入 `EventEmitter2`。

然后在类中如下使用它：

```typescript
this.eventEmitter.emit(
  'order.created',
  new OrderCreatedEvent({
    orderId: 1,
    payload: {},
  }),
);
```

#### 监听事件

要声明一个事件监听器，请在包含要执行代码的方法定义前使用 `@OnEvent()` 装饰器装饰该方法，如下所示：

```typescript
@OnEvent('order.created')
handleOrderCreatedEvent(payload: OrderCreatedEvent) {
  // 处理并处理 "OrderCreatedEvent" 事件
}
```

> warning **警告** 事件订阅者不能是请求作用域的。

第一个参数对于简单的事件发射器可以是 `string` 或 `symbol`，对于通配符发射器可以是 `string | symbol | Array<string | symbol>`。

第二个参数（可选）是一个监听器选项对象，如下所示：

```typescript
export type OnEventOptions = OnOptions & {
  /**
   * 如果为 "true"，则将给定的监听器预置（而不是追加）到监听器数组中。
   *
   * @see https://github.com/EventEmitter2/EventEmitter2#emitterprependlistenerevent-listener-options
   *
   * @default false
   */
  prependListener?: boolean;

  /**
   * 如果为 "true"，onEvent 回调在处理事件时不会抛出错误。否则，如果为 "false"，则会抛出错误。
   *
   * @default true
   */
  suppressErrors?: boolean;
};
```

> info **提示** 有关 `OnOptions` 选项对象的更多信息，请阅读 [`eventemitter2`](https://github.com/EventEmitter2/EventEmitter2#emitteronevent-listener-options-objectboolean)。

```typescript
@OnEvent('order.created', { async: true })
handleOrderCreatedEvent(payload: OrderCreatedEvent) {
  // 处理并处理 "OrderCreatedEvent" 事件
}
```

要使用命名空间/通配符，请将 `wildcard` 选项传递给 `EventEmitterModule#forRoot()` 方法。当启用命名空间/通配符时，事件可以是由分隔符分隔的字符串（`foo.bar`）或数组（`['foo', 'bar']`）。分隔符也可作为配置属性（`delimiter`）进行配置。启用命名空间功能后，您可以使用通配符订阅事件：

```typescript
@OnEvent('order.*')
handleOrderEvents(payload: OrderCreatedEvent | OrderRemovedEvent | OrderUpdatedEvent) {
  // 处理并处理事件
}
```

请注意，此类通配符仅适用于一个块。参数 `order.*` 将匹配例如事件 `order.created` 和 `order.shipped`，但不匹配 `order.delayed.out_of_stock`。为了监听此类事件，请使用 `EventEmitter2` [文档](https://github.com/EventEmitter2/EventEmitter2#multi-level-wildcards) 中描述的 `多级通配符` 模式（即 `**`）。

使用此模式，例如，您可以创建一个捕获所有事件的事件监听器。

```typescript
@OnEvent('**')
handleEverything(payload: any) {
  // 处理并处理事件
}
```

> info **提示** `EventEmitter2` 类提供了几种与事件交互的有用方法，例如 `waitFor` 和 `onAny`。您可以在 [这里](https://github.com/EventEmitter2/EventEmitter2) 阅读更多关于它们的信息。

#### 防止事件丢失

在 `onApplicationBootstrap` 生命周期钩子之前或期间触发的事件（例如来自模块构造函数或 `onModuleInit` 方法的事件）可能会被错过，因为 `EventSubscribersLoader` 可能尚未完成设置监听器。

为避免此问题，您可以使用 `EventEmitterReadinessWatcher` 的 `waitUntilReady` 方法，该方法返回一个在所有监听器注册完成后解析的 Promise。可以在模块的 `onApplicationBootstrap` 生命周期钩子中调用此方法，以确保所有事件都被正确捕获。

```typescript
await this.eventEmitterReadinessWatcher.waitUntilReady();
await this.eventEmitter.emit(
  'order.created',
  new OrderCreatedEvent({ orderId: 1, payload: {} }),
);
```

> info **注意** 这仅对于在 `onApplicationBootstrap` 生命周期钩子完成之前发出的事件是必需的。

#### 示例

一个可工作的示例可在 [这里](https://github.com/nestjs/nest/tree/master/sample/30-event-emitter) 找到。