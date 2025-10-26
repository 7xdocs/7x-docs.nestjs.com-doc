### MikroORM

本指南旨在帮助用户在 Nest 中开始使用 MikroORM。MikroORM 是基于数据映射器、工作单元和身份映射模式的 Node.js 的 TypeScript ORM。它是 TypeORM 的一个很好的替代品，从 TypeORM 迁移过来应该相当容易。MikroORM 的完整文档可以在[这里](https://mikro-orm.io/docs)找到。

> info **提示** `@mikro-orm/nestjs` 是一个第三方包，不由 NestJS 核心团队管理。请在[相应的代码仓库](https://github.com/mikro-orm/nestjs)报告该库的任何问题。

#### 安装

将 MikroORM 集成到 Nest 的最简单方法是通过 [`@mikro-orm/nestjs` 模块](https://github.com/mikro-orm/nestjs)。
只需在 Nest、MikroORM 和底层驱动旁边安装它：

```bash
$ npm i @mikro-orm/core @mikro-orm/nestjs @mikro-orm/sqlite
```

MikroORM 也支持 `postgres`、`sqlite` 和 `mongo`。查看[官方文档](https://mikro-orm.io/docs/usage-with-sql/)了解所有驱动。

安装过程完成后，我们可以将 `MikroOrmModule` 导入到根 `AppModule` 中。

```typescript
import { SqliteDriver } from '@mikro-orm/sqlite';

@Module({
  imports: [
    MikroOrmModule.forRoot({
      entities: ['./dist/entities'],
      entitiesTs: ['./src/entities'],
      dbName: 'my-db-name.sqlite3',
      driver: SqliteDriver,
    }),
  ],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {
}
```

`forRoot()` 方法接受与 MikroORM 包中的 `init()` 相同的配置对象。查看[此页面](https://mikro-orm.io/docs/configuration)获取完整的配置文档。

或者，我们可以通过创建配置文件 `mikro-orm.config.ts` 来[配置 CLI](https://mikro-orm.io/docs/installation#setting-up-the-commandline-tool)，然后不带任何参数调用 `forRoot()`。

```typescript
@Module({
  imports: [
    MikroOrmModule.forRoot(),
  ],
  ...
})
export class AppModule {}
```

但是当你使用使用 tree shaking 的构建工具时，这就不起作用了，为此最好显式提供配置：

```typescript
import config from './mikro-orm.config'; // 你的 ORM 配置

@Module({
  imports: [
    MikroOrmModule.forRoot(config),
  ],
  ...
})
export class AppModule {}
```

之后，`EntityManager` 就可以在整个项目中注入（而无需在其他任何地方导入任何模块）。

```ts
// 从你的驱动包或 `@mikro-orm/knex` 导入所有内容
import { EntityManager, MikroORM } from '@mikro-orm/sqlite';

@Injectable()
export class MyService {
  constructor(
    private readonly orm: MikroORM,
    private readonly em: EntityManager,
  ) {}
}
```

> info **提示** 注意 `EntityManager` 是从 `@mikro-orm/driver` 包导入的，其中 driver 是 `mysql`、`sqlite`、`postgres` 或你正在使用的任何驱动。如果你安装了 `@mikro-orm/knex` 作为依赖项，你也可以从那里导入 `EntityManager`。

#### 存储库

MikroORM 支持存储库设计模式。对于每个实体，我们可以创建一个存储库。在此处阅读关于存储库的完整文档。要定义哪些存储库应在当前范围中注册，你可以使用 `forFeature()` 方法。例如，通过以下方式：

> info **提示** 你**不应**通过 `forFeature()` 注册你的基础实体，因为这些实体没有存储库。另一方面，基础实体需要是 `forRoot()`（或一般 ORM 配置）列表中的一部分。

```typescript
// photo.module.ts
@Module({
  imports: [MikroOrmModule.forFeature([Photo])],
  providers: [PhotoService],
  controllers: [PhotoController],
})
export class PhotoModule {}
```

并将其导入到根 `AppModule` 中：

```typescript
// app.module.ts
@Module({
  imports: [MikroOrmModule.forRoot(...), PhotoModule],
})
export class AppModule {}
```

通过这种方式，我们可以使用 `@InjectRepository()` 装饰器将 `PhotoRepository` 注入到 `PhotoService` 中：

```typescript
@Injectable()
export class PhotoService {
  constructor(
    @InjectRepository(Photo)
    private readonly photoRepository: EntityRepository<Photo>,
  ) {}
}
```

#### 使用自定义存储库

当使用自定义存储库时，我们不再需要 `@InjectRepository()` 装饰器，因为 Nest 依赖注入基于类引用进行解析。

```ts
// `**./author.entity.ts**`
@Entity({ repository: () => AuthorRepository })
export class Author {
  // 为了在 `em.getRepository()` 中允许推断
  [EntityRepositoryType]?: AuthorRepository;
}

// `**./author.repository.ts**`
export class AuthorRepository extends EntityRepository<Author> {
  // 你的自定义方法...
}
```

由于自定义存储库名称与 `getRepositoryToken()` 返回的名称相同，我们不再需要 `@InjectRepository()` 装饰器：

```ts
@Injectable()
export class MyService {
  constructor(private readonly repo: AuthorRepository) {}
}
```

#### 自动加载实体

手动将实体添加到连接选项的实体数组中可能很繁琐。此外，从根模块引用实体破坏了应用程序领域边界，并导致实现细节泄漏到应用程序的其他部分。为了解决这个问题，可以使用静态 glob 路径。

但请注意，webpack 不支持 glob 路径，因此如果你在 monorepo 内构建应用程序，将无法使用它们。为了解决这个问题，提供了替代解决方案。要自动加载实体，请将配置对象（传递给 `forRoot()` 方法）的 `autoLoadEntities` 属性设置为 `true`，如下所示：

```ts
@Module({
  imports: [
    MikroOrmModule.forRoot({
      ...
      autoLoadEntities: true,
    }),
  ],
})
export class AppModule {}
```

指定该选项后，通过 `forFeature()` 方法注册的每个实体将自动添加到配置对象的实体数组中。

> info **提示** 注意，未通过 `forFeature()` 方法注册，而仅通过关系从实体引用的实体，不会通过 `autoLoadEntities` 设置包含在内。

> info **提示** 使用 `autoLoadEntities` 对 MikroORM CLI 也没有影响 - 为此我们仍然需要包含完整实体列表的 CLI 配置。另一方面，我们可以在那里使用 globs，因为 CLI 不会通过 webpack。

#### 序列化

> warning **注意** MikroORM 将每个实体关系包装在 `Reference<T>` 或 `Collection<T>` 对象中，以提供更好的类型安全性。这将使 [Nest 的内置序列化器](/techniques/serialization) 对所有包装的关系视而不见。换句话说，如果你从 HTTP 或 WebSocket 处理程序返回 MikroORM 实体，它们的所有关系都将**不会**被序列化。

幸运的是，MikroORM 提供了一个[序列化 API](https://mikro-orm.io/docs/serializing)，可以用来替代 `ClassSerializerInterceptor`。

```typescript
@Entity()
export class Book {
  @Property({ hidden: true }) // 相当于 class-transformer 的 `@Exclude`
  hiddenField = Date.now();

  @Property({ persist: false }) // 类似于 class-transformer 的 `@Expose()`。将仅存在于内存中，并将被序列化。
  count?: number;

  @ManyToOne({
    serializer: (value) => value.name,
    serializedName: 'authorName',
  }) // 相当于 class-transformer 的 `@Transform()`
  author: Author;
}
```

#### 队列中的请求范围处理程序

如[文档](https://mikro-orm.io/docs/identity-map)中所述，每个请求都需要一个干净的状态。这得益于通过中间件注册的 `RequestContext` 辅助器自动处理。

但是中间件仅针对常规 HTTP 请求处理执行，如果我们需要在此之外使用请求范围的方法怎么办？一个例子是队列处理程序或计划任务。

我们可以使用 `@CreateRequestContext()` 装饰器。它要求你首先将 `MikroORM` 实例注入到当前上下文中，然后它将用于为你创建上下文。在底层，该装饰器将为你的方法注册新的请求上下文，并在该上下文中执行它。

```ts
@Injectable()
export class MyService {
  constructor(private readonly orm: MikroORM) {}

  @CreateRequestContext()
  async doSomething() {
    // 这将在单独的上下文中执行
  }
}
```

> warning **注意** 顾名思义，此装饰器总是创建新的上下文，与其替代品 `@EnsureRequestContext` 相反，后者仅当尚未处于其他上下文中时才创建。

#### 测试

`@mikro-orm/nestjs` 包暴露了 `getRepositoryToken()` 函数，该函数返回基于给定实体准备的令牌，以允许模拟存储库。

```typescript
@Module({
  providers: [
    PhotoService,
    {
      // 或者当你有自定义存储库时：`provide: PhotoRepository`
      provide: getRepositoryToken(Photo), 
      useValue: mockedRepository,
    },
  ],
})
export class PhotoModule {}
```

#### 示例

可以在[这里](https://github.com/mikro-orm/nestjs-realworld-example-app)找到一个使用 NestJS 和 MikroORM 的真实世界示例。