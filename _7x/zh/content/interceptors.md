### 拦截器

拦截器是一个使用 `@Injectable()` 装饰器注解的类，并实现了 `NestInterceptor` 接口。

<figure><img class="illustrative-image" src="/assets/Interceptors_1.png" /></figure>

拦截器拥有一系列有用的能力，其灵感来自于 [面向切面编程](https://en.wikipedia.org/wiki/Aspect-oriented_programming) (AOP) 技术。它们使得以下功能成为可能：

- 在方法执行前后绑定额外的逻辑
- 转换从函数返回的结果
- 转换从函数抛出的异常
- 扩展基础函数行为
- 根据特定条件完全重写函数（例如，用于缓存目的）

#### 基础

每个拦截器实现 `intercept()` 方法，该方法接收两个参数。第一个是 `ExecutionContext` 实例（与 [守卫](/guards) 中的对象完全相同）。`ExecutionContext` 继承自 `ArgumentsHost`。我们在异常过滤器章节中之前见过 `ArgumentsHost`。在那里，我们看到它是围绕传递给原始处理程序的参数的包装器，并根据应用程序的类型包含不同的参数数组。你可以参考 [异常过滤器](https://docs.nestjs.com/exception-filters#arguments-host) 以了解更多关于这个主题的内容。

#### 执行上下文

通过扩展 `ArgumentsHost`，`ExecutionContext` 还添加了几个新的辅助方法，这些方法提供了关于当前执行过程的额外细节。这些细节有助于构建更通用的拦截器，这些拦截器可以在广泛的控制器、方法和执行上下文中工作。了解更多关于 `ExecutionContext` 的信息 [这里](/fundamentals/execution-context)。

#### 调用处理程序

第二个参数是一个 `CallHandler`。`CallHandler` 接口实现了 `handle()` 方法，你可以在拦截器中的某个时点使用它来调用路由处理方法。如果你在 `intercept()` 方法的实现中不调用 `handle()` 方法，那么路由处理方法将根本不会执行。

这种方法意味着 `intercept()` 方法有效地 **包装** 了请求/响应流。因此，你可以在最终路由处理程序执行 **之前和之后** 实现自定义逻辑。很明显，你可以在调用 `handle()` **之前** 在 `intercept()` 方法中编写代码，但是你如何影响之后发生的事情呢？因为 `handle()` 方法返回一个 `Observable`，我们可以使用强大的 [RxJS](https://github.com/ReactiveX/rxjs) 操作符来进一步操作响应。使用面向切面编程的术语，路由处理程序的调用（即调用 `handle()`）被称为 [切入点](https://en.wikipedia.org/wiki/Pointcut)，表明这是插入我们额外逻辑的点。

例如，考虑一个传入的 `POST /cats` 请求。该请求预定用于在 `CatsController` 内部定义的 `create()` 处理程序。如果沿途任何地方调用了不调用 `handle()` 方法的拦截器，那么 `create()` 方法将不会执行。一旦 `handle()` 被调用（并且其 `Observable` 已被返回），`create()` 处理程序将被触发。并且一旦通过 `Observable` 接收到响应流，就可以在流上执行额外的操作，并将最终结果返回给调用者。

<app-banner-devtools></app-banner-devtools>

#### 切面拦截

我们将看的第一个用例是使用拦截器来记录用户交互（例如，存储用户调用、异步分派事件或计算时间戳）。我们在下面展示一个简单的 `LoggingInterceptor`：

```typescript
@@filename(logging.interceptor)
// ... 代码保持不变
@@switch
// ... 代码保持不变
```

> info **提示** `NestInterceptor<T, R>` 是一个泛型接口，其中 `T` 表示 `Observable<T>` 的类型（支持响应流），而 `R` 是 `Observable<R>` 包装的值的类型。

> warning **注意** 拦截器，如控制器、提供者、守卫等，可以通过它们的 `constructor` **注入依赖**。

由于 `handle()` 返回一个 RxJS `Observable`，我们有很多操作符可以用来操作流。在上面的例子中，我们使用了 `tap()` 操作符，它在 observable 流正常或异常终止时调用我们的匿名日志记录函数，但不会以其他方式干扰响应周期。

#### 绑定拦截器

为了设置拦截器，我们使用从 `@nestjs/common` 包导入的 `@UseInterceptors()` 装饰器。像 [管道](/pipes) 和 [守卫](/guards) 一样，拦截器可以是控制器作用域、方法作用域或全局作用域。

```typescript
@@filename(cats.controller)
// ... 代码保持不变
```

> info **提示** `@UseInterceptors()` 装饰器是从 `@nestjs/common` 包导入的。

使用上述结构，`CatsController` 中定义的每个路由处理程序都将使用 `LoggingInterceptor`。当有人调用 `GET /cats` 端点时，你将在标准输出中看到以下输出：

```typescript
Before...
After... 1ms
```

注意，我们传递了 `LoggingInterceptor` 类（而不是实例），将实例化的责任留给框架，并启用依赖注入。与管道、守卫和异常过滤器一样，我们也可以传递一个就地实例：

```typescript
@@filename(cats.controller)
// ... 代码保持不变
```

如上所述，上述结构将拦截器附加到该控制器声明的每个处理程序上。如果我们想将拦截器的范围限制在单个方法上，我们只需在 **方法级别** 应用装饰器。

为了设置全局拦截器，我们使用 Nest 应用程序实例的 `useGlobalInterceptors()` 方法：

```typescript
const app = await NestFactory.create(AppModule);
app.useGlobalInterceptors(new LoggingInterceptor());
```

全局拦截器用于整个应用程序，针对每个控制器和每个路由处理程序。在依赖注入方面，从任何模块外部注册的全局拦截器（如上例中使用 `useGlobalInterceptors()`）无法注入依赖项，因为这是在任何模块的上下文之外完成的。为了解决这个问题，你可以使用以下结构 **直接从任何模块** 设置拦截器：

```typescript
@@filename(app.module)
// ... 代码保持不变
```

> info **提示** 当使用这种方法为拦截器执行依赖注入时，请注意，无论采用这种结构的模块是哪个，拦截器实际上都是全局的。应该在哪里完成这个操作？选择定义拦截器的模块（如上例中的 `LoggingInterceptor`）。此外，`useClass` 不是处理自定义提供者注册的唯一方法。了解更多 [这里](/fundamentals/custom-providers)。

#### 响应映射

我们已经知道 `handle()` 返回一个 `Observable`。流包含从路由处理程序 **返回** 的值，因此我们可以使用 RxJS 的 `map()` 操作符轻松地改变它。

> warning **警告** 响应映射功能不适用于库特定的响应策略（直接使用 `@Res()` 对象是被禁止的）。

让我们创建 `TransformInterceptor`，它将以一种简单的方式修改每个响应以演示该过程。它将使用 RxJS 的 `map()` 操作符将响应对象分配给新创建对象的 `data` 属性，将新对象返回给客户端。

```typescript
@@filename(transform.interceptor)
// ... 代码保持不变
@@switch
// ... 代码保持不变
```

> info **提示** Nest 拦截器同时支持同步和异步的 `intercept()` 方法。如果需要，你可以简单地将方法切换为 `async`。

使用上述结构，当有人调用 `GET /cats` 端点时，响应将如下所示（假设路由处理程序返回一个空数组 `[]`）：

```json
{
  "data": []
}
```

拦截器在为整个应用程序中出现的需求创建可重用的解决方案方面具有巨大价值。
例如，假设我们需要将每个出现的 `null` 值转换为空字符串 `''`。我们可以用一行代码完成这个操作，并全局绑定拦截器，以便它自动被每个注册的处理程序使用。

```typescript
@@filename()
// ... 代码保持不变
@@switch
// ... 代码保持不变
```

#### 异常映射

另一个有趣的用例是利用 RxJS 的 `catchError()` 操作符来覆盖抛出的异常：

```typescript
@@filename(errors.interceptor)
// ... 代码保持不变
@@switch
// ... 代码保持不变
```

#### 流覆盖

有几个原因有时我们可能希望完全阻止调用处理程序并返回一个不同的值。一个明显的例子是实现缓存以提高响应时间。让我们来看一个简单的 **缓存拦截器**，它从缓存返回其响应。在一个实际的例子中，我们需要考虑其他因素，如 TTL、缓存失效、缓存大小等，但这超出了本讨论的范围。这里我们将提供一个演示主要概念的基本示例。

```typescript
@@filename(cache.interceptor)
// ... 代码保持不变
@@switch
// ... 代码保持不变
```

我们的 `CacheInterceptor` 有一个硬编码的 `isCached` 变量和一个同样硬编码的响应 `[]`。需要注意的关键点是，我们在这里返回一个新的流，由 RxJS `of()` 操作符创建，因此路由处理程序 **根本不会被调用**。当有人调用使用 `CacheInterceptor` 的端点时，响应（一个硬编码的空数组）将立即返回。为了创建一个通用的解决方案，你可以利用 `Reflector` 并创建一个自定义装饰器。`Reflector` 在 [守卫](/guards) 章节中有详细描述。

#### 更多操作符

使用 RxJS 操作符操作流的可能性给了我们很多能力。让我们考虑另一个常见用例。假设你想处理路由请求的 **超时**。当你的端点在一段时间后没有返回任何内容时，你希望以错误响应终止。以下结构实现了这一点：

```typescript
@@filename(timeout.interceptor)
// ... 代码保持不变
@@switch
// ... 代码保持不变
```

5 秒后，请求处理将被取消。你还可以在抛出 `RequestTimeoutException` 之前添加自定义逻辑（例如释放资源）。