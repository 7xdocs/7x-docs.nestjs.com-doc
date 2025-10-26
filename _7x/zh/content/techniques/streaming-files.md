### 文件流传输

> info **注意** 本章节展示了如何从你的 **HTTP 应用程序** 中流式传输文件。下面展示的示例不适用于 GraphQL 或微服务应用程序。

有时你可能希望从 REST API 向客户端发送文件。在 Nest 中，通常你会这样做：

```ts
@Controller('file')
export class FileController {
  @Get()
  getFile(@Res() res: Response) {
    const file = createReadStream(join(process.cwd(), 'package.json'));
    file.pipe(res);
  }
}
```

但这样做，你将失去对控制器后拦截器逻辑的访问权限。为了解决这个问题，你可以返回一个 `StreamableFile` 实例，框架会在底层处理好响应的管道传输。

#### Streamable File 类

`StreamableFile` 是一个类，它持有要返回的流。要创建一个新的 `StreamableFile`，你可以将一个 `Buffer` 或 `Stream` 传递给 `StreamableFile` 构造函数。

> info **提示** `StreamableFile` 类可以从 `@nestjs/common` 导入。

#### 跨平台支持

默认情况下，Fastify 支持发送文件而无需调用 `stream.pipe(res)`，因此你完全不需要使用 `StreamableFile` 类。然而，Nest 在这两种平台类型中都支持使用 `StreamableFile`，所以如果你在 Express 和 Fastify 之间切换，无需担心两者引擎之间的兼容性问题。

#### 示例

下面是一个简单的示例，将 `package.json` 作为文件返回，而不是 JSON。但这种方法自然可以扩展到图片、文档和任何其他文件类型。

```ts
import { Controller, Get, StreamableFile } from '@nestjs/common';
import { createReadStream } from 'fs';
import { join } from 'path';

@Controller('file')
export class FileController {
  @Get()
  getFile(): StreamableFile {
    const file = createReadStream(join(process.cwd(), 'package.json'));
    return new StreamableFile(file);
  }
}
```

默认的内容类型（即 `Content-Type` HTTP 响应头的值）是 `application/octet-stream`。如果你需要自定义这个值，可以使用 `StreamableFile` 的 `type` 选项，或者使用 `res.set` 方法或 [`@Header()`](/controllers#headers) 装饰器，像这样：

```ts
import { Controller, Get, StreamableFile, Res } from '@nestjs/common';
import { createReadStream } from 'fs';
import { join } from 'path';
import type { Response } from 'express'; // 假设我们使用的是 ExpressJS HTTP 适配器

@Controller('file')
export class FileController {
  @Get()
  getFile(): StreamableFile {
    const file = createReadStream(join(process.cwd(), 'package.json'));
    return new StreamableFile(file, {
      type: 'application/json',
      disposition: 'attachment; filename="package.json"',
      // 如果你希望将 Content-Length 值定义为另一个值而不是文件的长度：
      // length: 123,
    });
  }

  // 或者甚至可以：
  @Get()
  getFileChangingResponseObjDirectly(@Res({ passthrough: true }) res: Response): StreamableFile {
    const file = createReadStream(join(process.cwd(), 'package.json'));
    res.set({
      'Content-Type': 'application/json',
      'Content-Disposition': 'attachment; filename="package.json"',
    });
    return new StreamableFile(file);
  }

  // 或者甚至可以：
  @Get()
  @Header('Content-Type', 'application/json')
  @Header('Content-Disposition', 'attachment; filename="package.json"')
  getFileUsingStaticValues(): StreamableFile {
    const file = createReadStream(join(process.cwd(), 'package.json'));
    return new StreamableFile(file);
  }  
}
```