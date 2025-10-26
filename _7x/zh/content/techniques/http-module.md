### HTTP 模块

[Axios](https://github.com/axios/axios) 是一个功能丰富的 HTTP 客户端包，被广泛使用。Nest 对 Axios 进行了封装，并通过内置的 `HttpModule` 将其暴露出来。`HttpModule` 导出了 `HttpService` 类，该类暴露了基于 Axios 的方法来执行 HTTP 请求。该库还将最终的 HTTP 响应转换为 `Observable`。

> info **提示** 你也可以直接使用任何通用目的的 Node.js HTTP 客户端库，包括 [got](https://github.com/sindresorhus/got) 或 [undici](https://github.com/nodejs/undici)。

#### 安装

要开始使用它，我们首先需要安装所需的依赖。

```bash
$ npm i --save @nestjs/axios axios
```

#### 入门

安装过程完成后，要使用 `HttpService`，首先导入 `HttpModule`。

```typescript
@Module({
  imports: [HttpModule],
  providers: [CatsService],
})
export class CatsModule {}
```

接下来，使用正常的构造函数注入方式注入 `HttpService`。

> info **提示** `HttpModule` 和 `HttpService` 是从 `@nestjs/axios` 包中导入的。

```typescript
@@filename()
@Injectable()
export class CatsService {
  constructor(private readonly httpService: HttpService) {}

  findAll(): Observable<AxiosResponse<Cat[]>> {
    return this.httpService.get('http://localhost:3000/cats');
  }
}
@@switch
@Injectable()
@Dependencies(HttpService)
export class CatsService {
  constructor(httpService) {
    this.httpService = httpService;
  }

  findAll() {
    return this.httpService.get('http://localhost:3000/cats');
  }
}
```

> info **提示** `AxiosResponse` 是从 `axios` 包 (`$ npm i axios`) 中导出的接口。

所有 `HttpService` 方法都返回一个包装在 `Observable` 对象中的 `AxiosResponse`。

#### 配置

[Axios](https://github.com/axios/axios) 可以通过多种选项进行配置，以自定义 `HttpService` 的行为。在[这里](https://github.com/axios/axios#request-config)阅读更多关于它们的信息。要配置底层的 Axios 实例，请在导入 `HttpModule` 时将一个可选的选项对象传递给 `HttpModule` 的 `register()` 方法。这个选项对象将直接传递给底层的 Axios 构造函数。

```typescript
@Module({
  imports: [
    HttpModule.register({
      timeout: 5000,
      maxRedirects: 5,
    }),
  ],
  providers: [CatsService],
})
export class CatsModule {}
```

#### 异步配置

当你需要异步传递模块选项而不是静态传递时，请使用 `registerAsync()` 方法。与大多数动态模块一样，Nest 提供了几种技术来处理异步配置。

一种技术是使用工厂函数：

```typescript
HttpModule.registerAsync({
  useFactory: () => ({
    timeout: 5000,
    maxRedirects: 5,
  }),
});
```

像其他工厂提供者一样，我们的工厂函数可以是[异步的](https://docs.nestjs.com/fundamentals/custom-providers#factory-providers-usefactory)，并且可以通过 `inject` 注入依赖。

```typescript
HttpModule.registerAsync({
  imports: [ConfigModule],
  useFactory: async (configService: ConfigService) => ({
    timeout: configService.get('HTTP_TIMEOUT'),
    maxRedirects: configService.get('HTTP_MAX_REDIRECTS'),
  }),
  inject: [ConfigService],
});
```

或者，你可以使用类而不是工厂来配置 `HttpModule`，如下所示。

```typescript
HttpModule.registerAsync({
  useClass: HttpConfigService,
});
```

上面的构造在 `HttpModule` 内部实例化 `HttpConfigService`，并使用它来创建一个选项对象。注意在这个例子中，`HttpConfigService` 必须实现 `HttpModuleOptionsFactory` 接口，如下所示。`HttpModule` 将在提供的类的实例化对象上调用 `createHttpOptions()` 方法。

```typescript
@Injectable()
class HttpConfigService implements HttpModuleOptionsFactory {
  createHttpOptions(): HttpModuleOptions {
    return {
      timeout: 5000,
      maxRedirects: 5,
    };
  }
}
```

如果你想要重用现有的选项提供者，而不是在 `HttpModule` 内部创建私有副本，请使用 `useExisting` 语法。

```typescript
HttpModule.registerAsync({
  imports: [ConfigModule],
  useExisting: HttpConfigService,
});
```

你也可以将所谓的 `extraProviders` 传递给 `registerAsync()` 方法。这些提供者将与模块提供者合并。

```typescript
HttpModule.registerAsync({
  imports: [ConfigModule],
  useClass: HttpConfigService,
  extraProviders: [MyAdditionalProvider],
});
```

当你想要向工厂函数或类构造函数提供额外的依赖时，这很有用。

#### 直接使用 Axios

如果你认为 `HttpModule.register` 的选项对你来说不够，或者你只是想访问由 `@nestjs/axios` 创建的底层 Axios 实例，你可以通过 `HttpService#axiosRef` 来访问它，如下所示：

```typescript
@Injectable()
export class CatsService {
  constructor(private readonly httpService: HttpService) {}

  findAll(): Promise<AxiosResponse<Cat[]>> {
    return this.httpService.axiosRef.get('http://localhost:3000/cats');
    //                      ^ AxiosInstance 接口
  }
}
```

#### 完整示例

由于 `HttpService` 方法的返回值是一个 Observable，我们可以使用 `rxjs` 的 `firstValueFrom` 或 `lastValueFrom` 来以 Promise 的形式获取请求的数据。

```typescript
import { catchError, firstValueFrom } from 'rxjs';

@Injectable()
export class CatsService {
  private readonly logger = new Logger(CatsService.name);
  constructor(private readonly httpService: HttpService) {}

  async findAll(): Promise<Cat[]> {
    const { data } = await firstValueFrom(
      this.httpService.get<Cat[]>('http://localhost:3000/cats').pipe(
        catchError((error: AxiosError) => {
          this.logger.error(error.response.data);
          throw 'An error happened!';
        }),
      ),
    );
    return data;
  }
}
```

> info **提示** 访问 RxJS 关于 [`firstValueFrom`](https://rxjs.dev/api/index/function/firstValueFrom) 和 [`lastValueFrom`](https://rxjs.dev/api/index/function/lastValueFrom) 的文档以了解它们之间的区别。