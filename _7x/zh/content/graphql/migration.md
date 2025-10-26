### 从 v10 迁移到 v11

本章提供了一组从 `@nestjs/graphql` 版本 10 迁移到版本 11 的指南。在此主要版本发布中，我们将 Apollo 驱动更新为与 Apollo Server v4（而不是 v3）兼容。注意：Apollo Server v4 中有一些破坏性变化（尤其是围绕插件和生态系统包），因此你必须相应地更新你的代码库。更多信息，请参阅 [Apollo Server v4 迁移指南](https://www.apollographql.com/docs/apollo-server/migration/)。

#### Apollo 包

不再安装 `apollo-server-express` 包，你需要安装 `@apollo/server`：

```bash
$ npm uninstall apollo-server-express
$ npm install @apollo/server
```

如果你使用 Fastify 适配器，你需要改为安装 `@as-integrations/fastify` 包：

```bash
$ npm uninstall apollo-server-fastify
$ npm install @apollo/server @as-integrations/fastify
```

#### Mercurius 包

Mercurius 网关不再是 `mercurius` 包的一部分。相反，你需要单独安装 `@mercuriusjs/gateway` 包：

```bash
$ npm install @mercuriusjs/gateway
```

类似地，为了创建联邦模式，你需要安装 `@mercuriusjs/federation` 包：

```bash
$ npm install @mercuriusjs/federation
```

### 从 v9 迁移到 v10

本章提供了一组从 `@nestjs/graphql` 版本 9 迁移到版本 10 的指南。这个主要版本发布的重点是提供一个更轻量级、与平台无关的核心库。

#### 引入 "driver" 包

在最新版本中，我们决定将 `@nestjs/graphql` 包拆分成几个独立的库，让你可以选择在项目中使用 Apollo (`@nestjs/apollo`)、Mercurius (`@nestjs/mercurius`) 还是其他 GraphQL 库。

这意味着现在你必须明确指定你的应用程序将使用什么驱动。

```typescript
// Before
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';

@Module({
  imports: [
    GraphQLModule.forRoot({
      autoSchemaFile: 'schema.gql',
    }),
  ],
})
export class AppModule {}

// After
import { ApolloDriver, ApolloDriverConfig } from '@nestjs/apollo';
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloDriverConfig>({
      driver: ApolloDriver,
      autoSchemaFile: 'schema.gql',
    }),
  ],
})
export class AppModule {}
```

#### 插件

Apollo Server 插件让你可以执行自定义操作以响应特定事件。由于这是 Apollo 的专属功能，我们将其从 `@nestjs/graphql` 移至新创建的 `@nestjs/apollo` 包中，因此你必须在应用程序中更新导入。

```typescript
// Before
import { Plugin } from '@nestjs/graphql';

// After
import { Plugin } from '@nestjs/apollo';
```

#### 指令

`schemaDirectives` 功能在 `@graphql-tools/schema` 包的 v8 版本中已被新的 [Schema directives API](https://www.graphql-tools.com/docs/schema-directives) 取代。

```typescript
// Before
import { SchemaDirectiveVisitor } from '@graphql-tools/utils';
import { defaultFieldResolver, GraphQLField } from 'graphql';

export class UpperCaseDirective extends SchemaDirectiveVisitor {
  visitFieldDefinition(field: GraphQLField<any, any>) {
    const { resolve = defaultFieldResolver } = field;
    field.resolve = async function (...args) {
      const result = await resolve.apply(this, args);
      if (typeof result === 'string') {
        return result.toUpperCase();
      }
      return result;
    };
  }
}

// After
import { getDirective, MapperKind, mapSchema } from '@graphql-tools/utils';
import { defaultFieldResolver, GraphQLSchema } from 'graphql';

export function upperDirectiveTransformer(
  schema: GraphQLSchema,
  directiveName: string,
) {
  return mapSchema(schema, {
    [MapperKind.OBJECT_FIELD]: (fieldConfig) => {
      const upperDirective = getDirective(
        schema,
        fieldConfig,
        directiveName,
      )?.[0];

      if (upperDirective) {
        const { resolve = defaultFieldResolver } = fieldConfig;

        // Replace the original resolver with a function that *first* calls
        // the original resolver, then converts its result to upper case
        fieldConfig.resolve = async function (source, args, context, info) {
          const result = await resolve(source, args, context, info);
          if (typeof result === 'string') {
            return result.toUpperCase();
          }
          return result;
        };
        return fieldConfig;
      }
    },
  });
}
```

要将此指令实现应用于包含 `@upper` 指令的模式，请使用 `transformSchema` 函数：

```typescript
GraphQLModule.forRoot<ApolloDriverConfig>({
  ...
  transformSchema: schema => upperDirectiveTransformer(schema, 'upper'),
})
```

#### 联邦

`GraphQLFederationModule` 已被移除，并替换为相应的驱动类：

```typescript
// Before
GraphQLFederationModule.forRoot({
  autoSchemaFile: true,
});

// After
GraphQLModule.forRoot<ApolloFederationDriverConfig>({
  driver: ApolloFederationDriver,
  autoSchemaFile: true,
});
```

> info **提示** `ApolloFederationDriver` 类和 `ApolloFederationDriverConfig` 都从 `@nestjs/apollo` 包中导出。

同样地，不再使用专用的 `GraphQLGatewayModule`，只需将相应的 `driver` 类传递给你的 `GraphQLModule` 设置：

```typescript
// Before
GraphQLGatewayModule.forRoot({
  gateway: {
    supergraphSdl: new IntrospectAndCompose({
      subgraphs: [
        { name: 'users', url: 'http://localhost:3000/graphql' },
        { name: 'posts', url: 'http://localhost:3001/graphql' },
      ],
    }),
  },
});

// After
GraphQLModule.forRoot<ApolloGatewayDriverConfig>({
  driver: ApolloGatewayDriver,
  gateway: {
    supergraphSdl: new IntrospectAndCompose({
      subgraphs: [
        { name: 'users', url: 'http://localhost:3000/graphql' },
        { name: 'posts', url: 'http://localhost:3001/graphql' },
      ],
    }),
  },
});
```

> info **提示** `ApolloGatewayDriver` 类和 `ApolloGatewayDriverConfig` 都从 `@nestjs/apollo` 包中导出。