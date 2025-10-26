### HTTPS

要创建一个使用HTTPS协议的应用程序，请在传递给`NestFactory`类的`create()`方法的选项对象中设置`httpsOptions`属性：

```typescript
const httpsOptions = {
  key: fs.readFileSync('./secrets/private-key.pem'),
  cert: fs.readFileSync('./secrets/public-certificate.pem'),
};
const app = await NestFactory.create(AppModule, {
  httpsOptions,
});
await app.listen(process.env.PORT ?? 3000);
```

如果你使用`FastifyAdapter`，请按如下方式创建应用程序：

```typescript
const app = await NestFactory.create<NestFastifyApplication>(
  AppModule,
  new FastifyAdapter({ https: httpsOptions }),
);
```

#### 同时运行多个服务器

以下方法展示了如何实例化一个同时监听多个端口（例如，一个非HTTPS端口和一个HTTPS端口）的Nest应用程序。

```typescript
const httpsOptions = {
  key: fs.readFileSync('./secrets/private-key.pem'),
  cert: fs.readFileSync('./secrets/public-certificate.pem'),
};

const server = express();
const app = await NestFactory.create(AppModule, new ExpressAdapter(server));
await app.init();

const httpServer = http.createServer(server).listen(3000);
const httpsServer = https.createServer(httpsOptions, server).listen(443);
```

因为我们自己调用了`http.createServer` / `https.createServer`，所以当调用`app.close`或收到终止信号时，NestJS不会关闭它们。我们需要自己来做这件事：

```typescript
@Injectable()
export class ShutdownObserver implements OnApplicationShutdown {
  private httpServers: http.Server[] = [];

  public addHttpServer(server: http.Server): void {
    this.httpServers.push(server);
  }

  public async onApplicationShutdown(): Promise<void> {
    await Promise.all(
      this.httpServers.map(
        (server) =>
          new Promise((resolve, reject) => {
            server.close((error) => {
              if (error) {
                reject(error);
              } else {
                resolve(null);
              }
            });
          }),
      ),
    );
  }
}

const shutdownObserver = app.get(ShutdownObserver);
shutdownObserver.addHttpServer(httpServer);
shutdownObserver.addHttpServer(httpsServer);
```

> info **提示** `ExpressAdapter` 是从 `@nestjs/platform-express` 包中导入的。`http` 和 `https` 包是Node.js的原生包。

> **警告** 此方法不适用于 [GraphQL 订阅](/graphql/subscriptions)。