### 异常过滤器

HTTP [异常过滤器](/exception-filters) 层与相应的 Web Sockets 层之间的唯一区别在于，你应该使用 `WsException` 而不是抛出 `HttpException`。

```typescript
throw new WsException('Invalid credentials.');
```

> info **提示** `WsException` 类是从 `@nestjs/websockets` 包中导入的。

使用上面的示例，Nest 将处理抛出的异常并发出具有以下结构的 `exception` 消息：

```typescript
{
  status: 'error',
  message: 'Invalid credentials.'
}
```

#### 过滤器

Web Sockets 异常过滤器的行为与 HTTP 异常过滤器等效。以下示例使用手动实例化的方法作用域过滤器。就像基于 HTTP 的应用程序一样，你也可以使用网关作用域的过滤器（例如，在网关类前添加 `@UseFilters()` 装饰器）。

```typescript
@UseFilters(new WsExceptionFilter())
@SubscribeMessage('events')
onEvent(client, data: any): WsResponse<any> {
  const event = 'events';
  return { event, data };
}
```

#### 继承

通常，你会创建完全自定义的异常过滤器，以满足你的应用程序需求。但是，在某些情况下，你可能希望简单地扩展**核心异常过滤器**，并根据特定因素覆盖其行为。

为了将异常处理委托给基础过滤器，你需要扩展 `BaseWsExceptionFilter` 并调用继承的 `catch()` 方法。

```typescript
@@filename()
import { Catch, ArgumentsHost } from '@nestjs/common';
import { BaseWsExceptionFilter } from '@nestjs/websockets';

@Catch()
export class AllExceptionsFilter extends BaseWsExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    super.catch(exception, host);
  }
}
@@switch
import { Catch } from '@nestjs/common';
import { BaseWsExceptionFilter } from '@nestjs/websockets';

@Catch()
export class AllExceptionsFilter extends BaseWsExceptionFilter {
  catch(exception, host) {
    super.catch(exception, host);
  }
}
```

上面的实现只是一个展示方法的框架。你扩展的异常过滤器的实现将包含你定制的**业务逻辑**（例如，处理各种条件）。