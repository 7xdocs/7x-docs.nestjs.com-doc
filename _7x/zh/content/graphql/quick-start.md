## 利用 TypeScript 和 GraphQL 的强大功能

[GraphQL](https://graphql.org/) 是一种强大的 API 查询语言，也是一个用于使用现有数据满足这些查询的运行时。它提供了一种优雅的方法，解决了 REST API 通常存在的许多问题。作为背景知识，我们建议阅读这篇关于 GraphQL 和 REST 的[比较](https://www.apollographql.com/blog/graphql-vs-rest)文章。GraphQL 与 [TypeScript](https://www.typescriptlang.org/) 结合使用，可以帮助您在 GraphQL 查询中实现更好的类型安全，为您提供端到端的类型支持。

在本章中，我们假设您对 GraphQL 有基本的了解，并重点介绍如何使用内置的 `@nestjs/graphql` 模块。`GraphQLModule` 可以配置为使用 [Apollo](https://www.apollographql.com/) 服务器（使用 `@nestjs/apollo` 驱动）和 [Mercurius](https://github.com/mercurius-js/mercurius)（使用 `@nestjs/mercurius` 驱动）。我们为这些成熟的 GraphQL 包提供了官方集成，以提供一种简单的方法来在 Nest 中使用 GraphQL（查看更多集成[这里](https://docs.nestjs.com/graphql/quick-start#third-party-integrations)）。

您也可以构建自己的专用驱动（在[这里](/graphql/other-features#creating-a-custom-driver)阅读更多信息）。

#### 安装

首先安装所需的包：

```bash
# 用于 Express 和 Apollo（默认）
$ npm i @nestjs/graphql @nestjs/apollo @apollo/server graphql

# 用于 Fastify 和 Apollo
# npm i @nestjs/graphql @nestjs/apollo @apollo/server @as-integrations/fastify graphql

# 用于 Fastify 和 Mercurius
# npm i @nestjs/graphql @nestjs/mercurius graphql mercurius
```

> warning **警告** `@nestjs/graphql@>=9` 和 `@nestjs/apollo^10` 包与 **Apollo v3** 兼容（查看 Apollo Server 3 [迁移指南](https://www.apollographql.com/docs/apollo-server/migration/) 了解更多详情），而 `@nestjs/graphql@^8` 仅支持 **Apollo v2**（例如，`apollo-server-express@2.x.x` 包）。

#### 概述

Nest 提供了两种构建 GraphQL 应用程序的方法：**代码优先**和**架构优先**方法。您应该选择最适合您的方法。本 GraphQL 部分中的大多数章节都分为两个主要部分：如果您采用**代码优先**方法，则应遵循一部分；如果您采用**架构优先**方法，则应使用另一部分。

在**代码优先**方法中，您使用装饰器和 TypeScript 类来生成相应的 GraphQL 架构。如果您希望仅使用 TypeScript 并避免在不同语言语法之间切换上下文，这种方法非常有用。

在**架构优先**方法中，真实来源是 GraphQL SDL（架构定义语言）文件。SDL 是一种与语言无关的方式，用于在不同平台之间共享架构文件。Nest 基于 GraphQL 架构自动生成您的 TypeScript 定义（使用类或接口），以减少编写冗余样板代码的需要。

<app-banner-courses-graphql-cf></app-banner-courses-graphql-cf>

#### GraphQL 和 TypeScript 入门

> info **提示** 在以下章节中，我们将集成 `@nestjs/apollo` 包。如果您想改用 `mercurius` 包，请导航到[此部分](/graphql/quick-start#mercurius-integration)。

安装包后，我们可以导入 `GraphQLModule` 并使用 `forRoot()` 静态方法进行配置。

```typescript
@@filename()
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';
import { ApolloDriver, ApolloDriverConfig } from '@nestjs/apollo';

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloDriverConfig>({
      driver: ApolloDriver,
    }),
  ],
})
export class AppModule {}
```

> info **提示** 对于 `mercurius` 集成，您应该使用 `MercuriusDriver` 和 `MercuriusDriverConfig`。两者都从 `@nestjs/mercurius` 包中导出。

`forRoot()` 方法接受一个选项对象作为参数。这些选项传递给底层驱动实例（在此处阅读有关可用设置的更多信息：[Apollo](https://www.apollographql.com/docs/apollo-server/v2/api/apollo-server.html#constructor-options-lt-ApolloServer-gt) 和 [Mercurius](https://github.com/mercurius-js/mercurius/blob/master/docs/api/options.md#plugin-options)）。例如，如果您想禁用 `playground` 并关闭 `debug` 模式（对于 Apollo），请传递以下选项：

```typescript
@@filename()
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';
import { ApolloDriver, ApolloDriverConfig } from '@nestjs/apollo';

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloDriverConfig>({
      driver: ApolloDriver,
      playground: false,
    }),
  ],
})
export class AppModule {}
```

在这种情况下，这些选项将转发给 `ApolloServer` 构造函数。

#### GraphQL playground

Playground 是一个图形化的、交互式的、浏览器内的 GraphQL IDE，默认情况下可在与 GraphQL 服务器相同的 URL 上使用。要访问 playground，您需要配置并运行一个基本的 GraphQL 服务器。要立即查看，您可以安装并构建[此处的工作示例](https://github.com/nestjs/nest/tree/master/sample/23-graphql-code-first)。或者，如果您正在按照这些代码示例进行操作，一旦完成[解析器章节](/graphql/resolvers-map)中的步骤，您就可以访问 playground。

完成这些步骤后，并且您的应用程序在后台运行，您可以打开 Web 浏览器并导航到 `http://localhost:3000/graphql`（主机和端口可能因您的配置而异）。然后您将看到 GraphQL playground，如下所示。

<figure>
  <img src="/assets/playground.png" alt="" />
</figure>

> warning **注意** `@nestjs/mercurius` 集成不附带内置的 GraphQL Playground 集成。相反，您可以使用 [GraphiQL](https://github.com/graphql/graphiql)（设置 `graphiql: true`）。

#### 多个端点

`@nestjs/graphql` 模块的另一个有用特性是能够同时服务多个端点。这使您可以决定哪些模块应包含在哪个端点中。默认情况下，`GraphQL` 在整个应用程序中搜索解析器。要将扫描限制为仅模块的子集，请使用 `include` 属性。

```typescript
GraphQLModule.forRoot({
  include: [CatsModule],
}),
```

> warning **警告** 如果您在单个应用程序中使用带有 `@as-integrations/fastify` 包的 `@apollo/server` 并具有多个 GraphQL 端点，请确保在 `GraphQLModule` 配置中启用 `disableHealthCheck` 设置。

#### 代码优先

在**代码优先**方法中，您使用装饰器和 TypeScript 类来生成相应的 GraphQL 架构。

要使用代码优先方法，首先将 `autoSchemaFile` 属性添加到选项对象：

```typescript
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  autoSchemaFile: join(process.cwd(), 'src/schema.gql'),
}),
```

`autoSchemaFile` 属性值是自动生成的架构将创建的位置。或者，可以在内存中动态生成架构。要启用此功能，请将 `autoSchemaFile` 属性设置为 `true`：

```typescript
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  autoSchemaFile: true,
}),
```

默认情况下，生成架构中的类型将按照它们在包含的模块中定义的顺序排列。要按字典顺序对架构进行排序，请将 `sortSchema` 属性设置为 `true`：

```typescript
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  autoSchemaFile: join(process.cwd(), 'src/schema.gql'),
  sortSchema: true,
}),
```

#### 示例

一个完整工作的代码优先示例可在[此处](https://github.com/nestjs/nest/tree/master/sample/23-graphql-code-first)获得。

#### 架构优先

要使用架构优先方法，首先将 `typePaths` 属性添加到选项对象。`typePaths` 属性指示 `GraphQLModule` 应在何处查找您将编写的 GraphQL SDL 架构定义文件。这些文件将在内存中组合；这允许您将架构拆分为多个文件并将其定位在解析器附近。

```typescript
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  typePaths: ['./**/*.graphql'],
}),
```

您通常还需要具有与 GraphQL SDL 类型对应的 TypeScript 定义（类和接口）。手动创建相应的 TypeScript 定义是冗余且繁琐的。这使我们没有单一的真实来源——在 SDL 中进行的每个更改都迫使我们同时调整 TypeScript 定义。为了解决这个问题，`@nestjs/graphql` 包可以**自动生成**来自抽象语法树（[AST](https://en.wikipedia.org/wiki/Abstract_syntax_tree)）的 TypeScript 定义。要启用此功能，请在配置 `GraphQLModule` 时添加 `definitions` 选项属性。

```typescript
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  typePaths: ['./**/*.graphql'],
  definitions: {
    path: join(process.cwd(), 'src/graphql.ts'),
  },
}),
```

`definitions` 对象的路径属性指示保存生成的 TypeScript 输出的位置。默认情况下，所有生成的 TypeScript 类型都创建为接口。要改为生成类，请使用值 `'class'` 指定 `outputAs` 属性。

```typescript
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  typePaths: ['./**/*.graphql'],
  definitions: {
    path: join(process.cwd(), 'src/graphql.ts'),
    outputAs: 'class',
  },
}),
```

上述方法在每次应用程序启动时动态生成 TypeScript 定义。或者，可能更倾向于构建一个简单的脚本以按需生成这些定义。例如，假设我们创建以下脚本作为 `generate-typings.ts`：

```typescript
import { GraphQLDefinitionsFactory } from '@nestjs/graphql';
import { join } from 'path';

const definitionsFactory = new GraphQLDefinitionsFactory();
definitionsFactory.generate({
  typePaths: ['./src/**/*.graphql'],
  path: join(process.cwd(), 'src/graphql.ts'),
  outputAs: 'class',
});
```

现在您可以按需运行此脚本：

```bash
$ ts-node generate-typings
```

> info **提示** 您可以预先编译脚本（例如，使用 `tsc`）并使用 `node` 执行它。

要为脚本启用监视模式（在任何 `.graphql` 文件更改时自动生成类型定义），请将 `watch` 选项传递给 `generate()` 方法。

```typescript
definitionsFactory.generate({
  typePaths: ['./src/**/*.graphql'],
  path: join(process.cwd(), 'src/graphql.ts'),
  outputAs: 'class',
  watch: true,
});
```

要为每个对象类型自动生成额外的 `__typename` 字段，请启用 `emitTypenameField` 选项。

```typescript
definitionsFactory.generate({
  // ...,
  emitTypenameField: true,
});
```

要将解析器（查询、变更、订阅）生成为没有参数的普通字段，请启用 `skipResolverArgs` 选项。

```typescript
definitionsFactory.generate({
  // ...,
  skipResolverArgs: true,
});
```

#### Apollo Sandbox

要使用 [Apollo Sandbox](https://www.apollographql.com/blog/announcement/platform/apollo-sandbox-an-open-graphql-ide-for-local-development/) 而不是 `graphql-playground` 作为本地开发的 GraphQL IDE，请使用以下配置：

```typescript
import { ApolloDriver, ApolloDriverConfig } from '@nestjs/apollo';
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';
import { ApolloServerPluginLandingPageLocalDefault } from '@apollo/server/plugin/landingPage/default';

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloDriverConfig>({
      driver: ApolloDriver,
      playground: false,
      plugins: [ApolloServerPluginLandingPageLocalDefault()],
    }),
  ],
})
export class AppModule {}
```

#### 示例

一个完整工作的架构优先示例可在[此处](https://github.com/nestjs/nest/tree/master/sample/12-graphql-schema-first)获得。

#### 访问生成的架构

在某些情况下（例如端到端测试），您可能希望获取对生成的架构对象的引用。在端到端测试中，您然后可以使用 `graphql` 对象运行查询，而无需使用任何 HTTP 监听器。

您可以使用 `GraphQLSchemaHost` 类访问生成的架构（在代码优先或架构优先方法中）：

```typescript
const { schema } = app.get(GraphQLSchemaHost);
```

> info **提示** 您必须在应用程序初始化后（在 `onModuleInit` 钩子被 `app.listen()` 或 `app.init()` 方法触发后）调用 `GraphQLSchemaHost#schema` getter。

#### 异步配置

当您需要异步传递模块选项而不是静态传递时，请使用 `forRootAsync()` 方法。与大多数动态模块一样，Nest 提供了几种技术来处理异步配置。

一种技术是使用工厂函数：

```typescript
 GraphQLModule.forRootAsync<ApolloDriverConfig>({
  driver: ApolloDriver,
  useFactory: () => ({
    typePaths: ['./**/*.graphql'],
  }),
}),
```

像其他工厂提供者一样，我们的工厂函数可以是 <a href="https://docs.nestjs.com/fundamentals/custom-providers#factory-providers-usefactory">异步的</a>，并且可以通过 `inject` 注入依赖项。

```typescript
GraphQLModule.forRootAsync<ApolloDriverConfig>({
  driver: ApolloDriver,
  imports: [ConfigModule],
  useFactory: async (configService: ConfigService) => ({
    typePaths: configService.get<string>('GRAPHQL_TYPE_PATHS'),
  }),
  inject: [ConfigService],
}),
```

或者，您可以使用类而不是工厂来配置 `GraphQLModule`，如下所示：

```typescript
GraphQLModule.forRootAsync<ApolloDriverConfig>({
  driver: ApolloDriver,
  useClass: GqlConfigService,
}),
```

上述构造在 `GraphQLModule` 内部实例化 `GqlConfigService`，使用它来创建选项对象。请注意，在此示例中，`GqlConfigService` 必须实现 `GqlOptionsFactory` 接口，如下所示。`GraphQLModule` 将在提供的类的实例化对象上调用 `createGqlOptions()` 方法。

```typescript
@Injectable()
class GqlConfigService implements GqlOptionsFactory {
  createGqlOptions(): ApolloDriverConfig {
    return {
      typePaths: ['./**/*.graphql'],
    };
  }
}
```

如果您想重用现有的选项提供者而不是在 `GraphQLModule` 内部创建私有副本，请使用 `useExisting` 语法。

```typescript
GraphQLModule.forRootAsync<ApolloDriverConfig>({
  imports: [ConfigModule],
  useExisting: ConfigService,
}),
```

#### Mercurius 集成

Fastify 用户（阅读更多[这里](/techniques/performance)）可以替代使用 `@nestjs/mercurius` 驱动而不是 Apollo。

```typescript
@@filename()
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';
import { MercuriusDriver, MercuriusDriverConfig } from '@nestjs/mercurius';

@Module({
  imports: [
    GraphQLModule.forRoot<MercuriusDriverConfig>({
      driver: MercuriusDriver,
      graphiql: true,
    }),
  ],
})
export class AppModule {}
```

> info **提示** 应用程序运行后，打开浏览器并导航到 `http://localhost:3000/graphiql`。您应该看到 [GraphQL IDE](https://github.com/graphql/graphiql)。

`forRoot()` 方法接受一个选项对象作为参数。这些选项传递给底层驱动实例。在此处阅读有关可用设置的更多信息[这里](https://github.com/mercurius-js/mercurius/blob/master/docs/api/options.md#plugin-options)。

#### 第三方集成

- [GraphQL Yoga](https://github.com/dotansimha/graphql-yoga)

#### 示例

一个工作示例可在[此处](https://github.com/nestjs/nest/tree/master/sample/33-graphql-mercurius)获得。