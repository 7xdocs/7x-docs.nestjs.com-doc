### 保持活动连接

默认情况下，NestJS的HTTP适配器会等待响应完成后再关闭应用程序。但有时，这种行为并不理想，或者出乎意料。可能有些请求使用`Connection: Keep-Alive`头，这些请求会持续很长时间。

对于这些你希望应用程序不等待请求结束就退出的场景，你可以在创建NestJS应用程序时启用`forceCloseConnections`选项。

> warning **Tip** 大多数用户不需要启用此选项。但需要此选项的迹象是，你的应用程序没有按预期退出。通常当启用`app.enableShutdownHooks()`时，你会发现应用程序没有重启/退出。这最有可能发生在开发过程中使用`--watch`运行NestJS应用程序时。

#### 使用方法

在你的`main.ts`文件中，创建NestJS应用程序时启用此选项：

```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule, {
    forceCloseConnections: true,
  });
  await app.listen(process.env.PORT ?? 3000);
}

bootstrap();
```