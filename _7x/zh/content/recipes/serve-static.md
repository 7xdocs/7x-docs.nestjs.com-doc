### Serve Static

为了提供静态内容（如单页应用 SPA），我们可以使用 [`@nestjs/serve-static`](https://www.npmjs.com/package/@nestjs/serve-static) 包中的 `ServeStaticModule`。

#### 安装

首先我们需要安装必需的包：

```bash
$ npm install --save @nestjs/serve-static
```

#### 引导

安装过程完成后，我们可以将 `ServeStaticModule` 导入到根模块 `AppModule` 中，并通过向 `forRoot()` 方法传递一个配置对象来配置它。

```typescript
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { ServeStaticModule } from '@nestjs/serve-static';
import { join } from 'path';

@Module({
  imports: [
    ServeStaticModule.forRoot({
      rootPath: join(__dirname, '..', 'client'),
    }),
  ],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

完成此配置后，构建静态网站并将其内容放置在 `rootPath` 属性指定的位置。

#### 配置

`ServeStaticModule` 可以通过多种选项进行配置，以自定义其行为。
你可以设置渲染静态应用的路径，指定排除的路径，启用或禁用设置 Cache-Control 响应头等。查看完整的选项列表请[点击这里](https://github.com/nestjs/serve-static/blob/master/lib/interfaces/serve-static-options.interface.ts)。

> warning **注意** 静态应用的默认 `renderPath` 是 `*`（所有路径），并且该模块将响应发送 "index.html" 文件。
> 这使你可以为 SPA 创建客户端路由。控制器中指定的路径将回退到服务器。
> 你可以通过设置 `serveRoot`、`renderPath` 并结合其他选项来更改此行为。

#### 示例

一个可用的示例在[这里](https://github.com/nestjs/nest/tree/master/sample/24-serve-static)。