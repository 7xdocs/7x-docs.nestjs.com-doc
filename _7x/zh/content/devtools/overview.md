### 概述

> info **提示** 本章介绍 Nest Devtools 与 Nest 框架的集成。如果您正在寻找 Devtools 应用程序，请访问 [Devtools](https://devtools.nestjs.com) 网站。

要开始调试本地应用程序，请打开 `main.ts` 文件，并确保在应用程序选项对象中将 `snapshot` 属性设置为 `true`，如下所示：

```typescript
async function bootstrap() {
  const app = await NestFactory.create(AppModule, {
    snapshot: true,
  });
  await app.listen(process.env.PORT ?? 3000);
}
```

这将指示框架收集必要的元数据，以便 Nest Devtools 可视化您的应用程序图。

接下来，让我们安装所需的依赖项：

```bash
$ npm i @nestjs/devtools-integration
```

> warning **警告** 如果您的应用程序中使用了 `@nestjs/graphql` 包，请确保安装最新版本（`npm i @nestjs/graphql@11`）。

安装好这个依赖项后，打开 `app.module.ts` 文件并导入我们刚刚安装的 `DevtoolsModule`：

```typescript
@Module({
  imports: [
    DevtoolsModule.register({
      http: process.env.NODE_ENV !== 'production',
    }),
  ],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

> warning **警告** 我们在这里检查 `NODE_ENV` 环境变量的原因是，您绝不应在生产环境中使用此模块！

一旦导入了 `DevtoolsModule` 并且您的应用程序已启动运行（`npm run start:dev`），您应该能够导航到 [Devtools](https://devtools.nestjs.com) 网址并看到 introspected 图。

<figure><img src="/assets/devtools/modules-graph.png" /></figure>

> info **提示** 正如您在上面的截图中看到的，每个模块都连接到 `InternalCoreModule`。`InternalCoreModule` 是一个全局模块，总是会导入到根模块中。由于它被注册为全局节点，Nest 会自动在所有模块和 `InternalCoreModule` 节点之间创建连接。现在，如果您想从图中隐藏全局模块，可以使用（侧边栏中的）"**隐藏全局模块**"复选框。

由此可见，`DevtoolsModule` 使您的应用程序暴露一个额外的 HTTP 服务器（在 8000 端口上），Devtools 应用程序将使用该服务器来 introspect 您的应用。

为了再次确认一切正常工作，将图视图切换到“Classes”。您应该会看到以下屏幕：

<figure><img src="/assets/devtools/classes-graph.png" /></figure>

要聚焦于特定节点，请点击该矩形，图将显示一个带有“聚焦”按钮的弹出窗口。您也可以使用（位于侧边栏中的）搜索栏来查找特定节点。

> info **提示** 如果您点击“检查”按钮，应用程序将带您进入 `/debug` 页面，并选中该特定节点。

<figure><img src="/assets/devtools/node-popup.png" /></figure>

> info **提示** 要将图导出为图像，请点击图右上角的“导出为 PNG”按钮。

使用位于（左侧）侧边栏中的表单控件，您可以控制连接的接近程度，例如，可视化特定的应用程序子树：

<figure><img src="/assets/devtools/subtree-view.png" /></figure>

当您的团队中有**新开发人员**，并且您想向他们展示应用程序的结构时，这会特别有用。您也可以使用此功能来可视化特定模块（例如 `TasksModule`）及其所有依赖项，这在您将大型应用程序拆分为较小的模块（例如，单个微服务）时会很方便。

您可以观看此视频，了解“图浏览器”功能的实际应用：

<figure>
  <iframe
    width="1000"
    height="565"
    src="https://www.youtube.com/embed/bW8V-ssfnvM"
    title="YouTube video player"
    frameBorder="0"
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
    allowFullScreen
  ></iframe>
</figure>

#### 调查“无法解析依赖项”错误

> info **注意** 此功能支持 `@nestjs/core` >= `v9.3.10`。

您可能见过的最常见的错误消息之一是关于 Nest 无法解析提供者的依赖项。使用 Nest Devtools，您可以轻松识别问题并了解如何解决它。

首先，打开 `main.ts` 文件并更新 `bootstrap()` 调用，如下所示：

```typescript
bootstrap().catch((err) => {
  fs.writeFileSync('graph.json', PartialGraphHost.toString() ?? '');
  process.exit(1);
});
```

同时，确保将 `abortOnError` 设置为 `false`：

```typescript
const app = await NestFactory.create(AppModule, {
  snapshot: true,
  abortOnError: false, // <--- 这里
});
```

现在，每当您的应用程序因“无法解析依赖项”错误而启动失败时，您会在根目录中找到 `graph.json`（代表部分图）文件。然后，您可以将此文件拖放到 Devtools 中（确保将当前模式从“交互式”切换到“预览”）：

<figure><img src="/assets/devtools/drag-and-drop.png" /></figure>

成功上传后，您应该会看到以下图和对话框窗口：

<figure><img src="/assets/devtools/partial-graph-modules-view.png" /></figure>

如您所见，高亮显示的 `TasksModule` 是我们应该查看的模块。此外，在对话框窗口中，您已经可以看到一些关于如何修复此问题的说明。

如果我们切换到“Classes”视图，会看到以下内容：

<figure><img src="/assets/devtools/partial-graph-classes-view.png" /></figure>

此图表明，我们想要注入到 `TasksService` 中的 `DiagnosticsService` 在 `TasksModule` 模块的上下文中未找到，我们可能只需将 `DiagnosticsModule` 导入到 `TasksModule` 模块中即可解决此问题！

#### 路由浏览器

当您导航到“路由浏览器”页面时，您应该会看到所有已注册的入口点：

<figure><img src="/assets/devtools/routes.png" /></figure>

> info **提示** 此页面不仅显示 HTTP 路由，还显示所有其他入口点（例如 WebSockets、gRPC、GraphQL 解析器等）。

入口点按其宿主控制器分组。您也可以使用搜索栏查找特定的入口点。

如果您点击特定的入口点，将显示**流程图**。此图显示入口点的执行流程（例如，绑定到此路由的守卫、拦截器、管道等）。当您想了解特定路由的请求/响应周期是什么样的，或者排查为什么特定的守卫/拦截器/管道没有被执行时，这特别有用。

#### 沙箱

要实时执行 JavaScript 代码并与您的应用程序交互，请导航到“沙箱”页面：

<figure><img src="/assets/devtools/sandbox.png" /></figure>

该 playground 可用于**实时**测试和调试 API 端点，使开发人员能够快速识别和修复问题，而无需使用例如 HTTP 客户端。我们还可以绕过身份验证层，因此不再需要额外的登录步骤，甚至不需要专门的测试用户账户。对于事件驱动的应用程序，我们也可以直接从 playground 触发事件，并查看应用程序的反应。

所有记录的内容都会流转到 playground 的控制台，因此我们可以轻松了解正在发生的事情。

只需**实时**执行代码并立即查看结果，而无需重新构建应用程序和重启服务器。

<figure><img src="/assets/devtools/sandbox-table.png" /></figure>

> info **提示** 要美观地显示对象数组，请使用 `console.table()`（或直接使用 `table()`）函数。

您可以观看此视频，了解“交互式 Playground”功能的实际应用：

<figure>
  <iframe
    width="1000"
    height="565"
    src="https://www.youtube.com/embed/liSxEN_VXKM"
    title="YouTube video player"
    frameBorder="0"
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
    allowFullScreen
  ></iframe>
</figure>

#### 启动性能分析器

要查看所有类节点（控制器、提供者、增强器等）及其相应的实例化时间，请导航到“启动性能”页面：

<figure><img src="/assets/devtools/bootstrap-performance.png" /></figure>

当您想识别应用程序启动过程中最慢的部分时（例如，当您想优化应用程序的启动时间时，这对于例如无服务器环境至关重要），此页面特别有用。

#### 审计

要查看应用程序在分析序列化图时自动生成的审计（错误/警告/提示），请导航到“审计”页面：

<figure><img src="/assets/devtools/audit.png" /></figure>

> info **提示** 上面的截图并未显示所有可用的审计规则。

当您想识别应用程序中的潜在问题时，此页面会很有用。

#### 预览静态文件

要将序列化图保存到文件，请使用以下代码：

```typescript
await app.listen(process.env.PORT ?? 3000); // 或 await app.init()
fs.writeFileSync('./graph.json', app.get(SerializedGraph).toString());
```

> info **提示** `SerializedGraph` 从 `@nestjs/core` 包导出。

然后您可以拖放/上传此文件：

<figure><img src="/assets/devtools/drag-and-drop.png" /></figure>

当您想与其他人（例如同事）共享您的图，或者想离线分析它时，这会很有帮助。