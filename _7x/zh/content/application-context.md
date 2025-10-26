### 独立应用程序

有多种挂载 Nest 应用程序的方式。你可以创建 Web 应用、微服务，或者只是一个纯粹的 Nest **独立应用程序**（不包含任何网络监听器）。Nest 独立应用程序是 Nest **IoC 容器**的包装器，它持有所有已实例化的类。我们可以使用独立应用程序对象直接从任何导入的模块中获取任何现有实例的引用。因此，你可以在任何地方利用 Nest 框架，例如，包括脚本化的 **CRON** 作业。你甚至可以在其上构建 **CLI**。

#### 入门

要创建 Nest 独立应用程序，请使用以下结构：

```typescript
@@filename()
async function bootstrap() {
  const app = await NestFactory.createApplicationContext(AppModule);
  // 在此处编写你的应用程序逻辑...
}
bootstrap();
```

#### 从静态模块中获取提供者

独立应用程序对象允许你获取在 Nest 应用程序中注册的任何实例的引用。假设我们在被 `AppModule` 模块导入的 `TasksModule` 模块中有一个 `TasksService` 提供者。这个类提供了一组我们希望从 CRON 作业内部调用的方法。

```typescript
@@filename()
const tasksService = app.get(TasksService);
```

为了访问 `TasksService` 实例，我们使用 `get()` 方法。`get()` 方法就像一个**查询**，它在每个已注册的模块中搜索实例。你可以向其传递任何提供者的令牌。或者，为了进行严格的上下文检查，可以传递一个包含 `strict: true` 属性的选项对象。启用此选项后，你必须通过特定的模块来从选定的上下文中获取特定的实例。

```typescript
@@filename()
const tasksService = app.select(TasksModule).get(TasksService, { strict: true });
```

以下是可用于从独立应用程序对象中获取实例引用的方法摘要。

<table>
  <tr>
    <td>
      <code>get()</code>
    </td>
    <td>
      获取应用程序上下文中可用的控制器或提供者（包括守卫、过滤器等）的实例。
    </td>
  </tr>
  <tr>
    <td>
      <code>select()</code>
    </td>
    <td>
      在模块图中导航，以从所选模块中取出特定的实例（与上述严格模式一起使用）。
    </td>
  </tr>
</table>

> info **提示** 在非严格模式下，默认选择根模块。要选择任何其他模块，你需要手动逐步导航模块图。

请注意，独立应用程序没有任何网络监听器，因此任何与 HTTP 相关的 Nest 功能（例如，中间件、拦截器、管道、守卫等）在此上下文中都不可用。

例如，即使你在应用程序中注册了全局拦截器，然后使用 `app.get()` 方法获取控制器的实例，该拦截器也不会被执行。

#### 从动态模块中获取提供者

在处理[动态模块](./fundamentals/dynamic-modules.md)时，我们需要向 `app.select` 提供与应用程序中注册的动态模块相同的对象。例如：

```typescript
@@filename()
export const dynamicConfigModule = ConfigModule.register({ folder: './config' });

@Module({
  imports: [dynamicConfigModule],
})
export class AppModule {}
```

然后你可以稍后选择该模块：

```typescript
@@filename()
const configService = app.select(dynamicConfigModule).get(ConfigService, { strict: true });
```

#### 终止阶段

如果你希望 Node 应用程序在脚本运行结束后关闭（例如，对于运行 CRON 作业的脚本），你必须在 `bootstrap` 函数的末尾调用 `app.close()` 方法，如下所示：

```typescript
@@filename()
async function bootstrap() {
  const app = await NestFactory.createApplicationContext(AppModule);
  // 应用程序逻辑...
  await app.close();
}
bootstrap();
```

正如在[生命周期事件](./fundamentals/lifecycle-events.md)章节中提到的，这将触发生命周期钩子。

#### 示例

一个可用的示例在[这里](https://github.com/nestjs/nest/tree/master/sample/18-context)。