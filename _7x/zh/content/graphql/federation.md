### 联合（Federation）

联合（Federation）提供了一种将单体 GraphQL 服务器拆分为独立微服务的方法。它由两个组件组成：一个网关和一个或多个联合微服务。每个微服务持有部分模式（schema），网关将这些模式合并成一个可供客户端使用的单一模式。

引用 [Apollo 文档](https://blog.apollographql.com/apollo-federation-f260cf525d21) 的说法，Federation 的设计遵循以下核心原则：

- 构建图（graph）应该是**声明式**的。通过 federation，您可以在模式内部声明式地组合图，而不是编写命令式的模式拼接代码。
- 代码应按**关注点**分离，而不是按类型分离。通常，没有一个团队能够控制像 User 或 Product 这样的重要类型的每个方面，因此这些类型的定义应该分布在团队和代码库中，而不是集中管理。
- 图应该便于客户端使用。联合服务可以共同形成一个完整的、以产品为中心的图，准确反映客户端如何使用它。
- 它就是 **GraphQL**，仅使用语言规范中的特性。任何语言，不仅仅是 JavaScript，都可以实现 federation。

> warning **警告** Federation 目前不支持订阅（subscriptions）。

在以下部分中，我们将设置一个由网关和两个联合端点组成的演示应用程序：用户服务和帖子服务。

#### 使用 Apollo 进行 Federation

首先安装所需的依赖：

```bash
$ npm install --save @apollo/subgraph
```

#### 模式优先（Schema first）

"用户服务"提供了一个简单的模式。注意 `@key` 指令：它指示 Apollo 查询规划器，如果指定了 `id`，则可以获取特定的 `User` 实例。另外，请注意我们 `extend` 了 `Query` 类型。

```graphql
type User @key(fields: "id") {
  id: ID!
  name: String!
}

extend type Query {
  getUser(id: ID!): User
}
```

解析器提供了一个名为 `resolveReference()` 的额外方法。每当相关资源需要 User 实例时，Apollo Gateway 就会触发该方法。我们稍后将在帖子服务中看到一个例子。请注意，该方法必须使用 `@ResolveReference()` 装饰器进行注解。

```typescript
import { Args, Query, Resolver, ResolveReference } from '@nestjs/graphql';
import { UsersService } from './users.service';

@Resolver('User')
export class UsersResolver {
  constructor(private usersService: UsersService) {}

  @Query()
  getUser(@Args('id') id: string) {
    return this.usersService.findById(id);
  }

  @ResolveReference()
  resolveReference(reference: { __typename: string; id: string }) {
    return this.usersService.findById(reference.id);
  }
}
```

最后，我们通过注册 `GraphQLModule` 并在配置对象中传入 `ApolloFederationDriver` 驱动来将所有内容连接起来：

```typescript
import {
  ApolloFederationDriver,
  ApolloFederationDriverConfig,
} from '@nestjs/apollo';
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';
import { UsersResolver } from './users.resolver';

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloFederationDriverConfig>({
      driver: ApolloFederationDriver,
      typePaths: ['**/*.graphql'],
    }),
  ],
  providers: [UsersResolver],
})
export class AppModule {}
```

#### 代码优先（Code first）

首先向 `User` 实体添加一些额外的装饰器。

```ts
import { Directive, Field, ID, ObjectType } from '@nestjs/graphql';

@ObjectType()
@Directive('@key(fields: "id")')
export class User {
  @Field(() => ID)
  id: number;

  @Field()
  name: string;
}
```

解析器提供了一个名为 `resolveReference()` 的额外方法。每当相关资源需要 User 实例时，Apollo Gateway 就会触发该方法。我们稍后将在帖子服务中看到一个例子。请注意，该方法必须使用 `@ResolveReference()` 装饰器进行注解。

```ts
import { Args, Query, Resolver, ResolveReference } from '@nestjs/graphql';
import { User } from './user.entity';
import { UsersService } from './users.service';

@Resolver(() => User)
export class UsersResolver {
  constructor(private usersService: UsersService) {}

  @Query(() => User)
  getUser(@Args('id') id: number): User {
    return this.usersService.findById(id);
  }

  @ResolveReference()
  resolveReference(reference: { __typename: string; id: number }): User {
    return this.usersService.findById(reference.id);
  }
}
```

最后，我们通过注册 `GraphQLModule` 并在配置对象中传入 `ApolloFederationDriver` 驱动来将所有内容连接起来：

```typescript
import {
  ApolloFederationDriver,
  ApolloFederationDriverConfig,
} from '@nestjs/apollo';
import { Module } from '@nestjs/common';
import { UsersResolver } from './users.resolver';
import { UsersService } from './users.service'; // 此示例中未包含

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloFederationDriverConfig>({
      driver: ApolloFederationDriver,
      autoSchemaFile: true,
    }),
  ],
  providers: [UsersResolver, UsersService],
})
export class AppModule {}
```

在代码优先模式下，可用的工作示例在[这里](https://github.com/nestjs/nest/tree/master/sample/31-graphql-federation-code-first/users-application)，在模式优先模式下在[这里](https://github.com/nestjs/nest/tree/master/sample/32-graphql-federation-schema-first/users-application)。

#### 联合示例：帖子（Posts）

帖子服务旨在通过 `getPosts` 查询提供聚合的帖子，同时也使用 `user.posts` 字段扩展我们的 `User` 类型。

#### 模式优先（Schema first）

"帖子服务"在其模式中通过使用 `extend` 关键字引用 `User` 类型。它还在 `User` 类型上声明了一个额外的属性（`posts`）。注意用于匹配 User 实例的 `@key` 指令，以及指示 `id` 字段在其他地方管理的 `@external` 指令。

```graphql
type Post @key(fields: "id") {
  id: ID!
  title: String!
  body: String!
  user: User
}

extend type User @key(fields: "id") {
  id: ID! @external
  posts: [Post]
}

extend type Query {
  getPosts: [Post]
}
```

在以下示例中，`PostsResolver` 提供了 `getUser()` 方法，该方法返回一个包含 `__typename` 和您的应用程序可能需要解析引用的其他属性的引用，在这个例子中是 `id`。GraphQL Gateway 使用 `__typename` 来定位负责 User 类型的微服务并检索相应的实例。执行 `resolveReference()` 方法时，将请求上述的"用户服务"。

```typescript
import { Query, Resolver, Parent, ResolveField } from '@nestjs/graphql';
import { PostsService } from './posts.service';
import { Post } from './posts.interfaces';

@Resolver('Post')
export class PostsResolver {
  constructor(private postsService: PostsService) {}

  @Query('getPosts')
  getPosts() {
    return this.postsService.findAll();
  }

  @ResolveField('user')
  getUser(@Parent() post: Post) {
    return { __typename: 'User', id: post.userId };
  }
}
```

最后，我们必须注册 `GraphQLModule`，类似于我们在"用户服务"部分所做的。

```typescript
import {
  ApolloFederationDriver,
  ApolloFederationDriverConfig,
} from '@nestjs/apollo';
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';
import { PostsResolver } from './posts.resolver';

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloFederationDriverConfig>({
      driver: ApolloFederationDriver,
      typePaths: ['**/*.graphql'],
    }),
  ],
  providers: [PostsResolvers],
})
export class AppModule {}
```

#### 代码优先（Code first）

首先，我们必须声明一个代表 `User` 实体的类。尽管实体本身存在于另一个服务中，但我们将在这里使用它（扩展其定义）。注意 `@extends` 和 `@external` 指令。

```ts
import { Directive, ObjectType, Field, ID } from '@nestjs/graphql';
import { Post } from './post.entity';

@ObjectType()
@Directive('@extends')
@Directive('@key(fields: "id")')
export class User {
  @Field(() => ID)
  @Directive('@external')
  id: number;

  @Field(() => [Post])
  posts?: Post[];
}
```

现在让我们为 `User` 实体上的扩展创建相应的解析器，如下所示：

```ts
import { Parent, ResolveField, Resolver } from '@nestjs/graphql';
import { PostsService } from './posts.service';
import { Post } from './post.entity';
import { User } from './user.entity';

@Resolver(() => User)
export class UsersResolver {
  constructor(private readonly postsService: PostsService) {}

  @ResolveField(() => [Post])
  public posts(@Parent() user: User): Post[] {
    return this.postsService.forAuthor(user.id);
  }
}
```

我们还必须定义 `Post` 实体类：

```ts
import { Directive, Field, ID, Int, ObjectType } from '@nestjs/graphql';
import { User } from './user.entity';

@ObjectType()
@Directive('@key(fields: "id")')
export class Post {
  @Field(() => ID)
  id: number;

  @Field()
  title: string;

  @Field(() => Int)
  authorId: number;

  @Field(() => User)
  user?: User;
}
```

及其解析器：

```ts
import { Query, Args, ResolveField, Resolver, Parent } from '@nestjs/graphql';
import { PostsService } from './posts.service';
import { Post } from './post.entity';
import { User } from './user.entity';

@Resolver(() => Post)
export class PostsResolver {
  constructor(private readonly postsService: PostsService) {}

  @Query(() => Post)
  findPost(@Args('id') id: number): Post {
    return this.postsService.findOne(id);
  }

  @Query(() => [Post])
  getPosts(): Post[] {
    return this.postsService.all();
  }

  @ResolveField(() => User)
  user(@Parent() post: Post): any {
    return { __typename: 'User', id: post.authorId };
  }
}
```

最后，在一个模块中将所有内容连接起来。注意模式构建选项，我们在其中指定 `User` 是一个孤立（外部）类型。

```ts
import {
  ApolloFederationDriver,
  ApolloFederationDriverConfig,
} from '@nestjs/apollo';
import { Module } from '@nestjs/common';
import { User } from './user.entity';
import { PostsResolvers } from './posts.resolvers';
import { UsersResolvers } from './users.resolvers';
import { PostsService } from './posts.service'; // 此示例中未包含

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloFederationDriverConfig>({
      driver: ApolloFederationDriver,
      autoSchemaFile: true,
      buildSchemaOptions: {
        orphanedTypes: [User],
      },
    }),
  ],
  providers: [PostsResolver, UsersResolver, PostsService],
})
export class AppModule {}
```

在代码优先模式下，可用的工作示例在[这里](https://github.com/nestjs/nest/tree/master/sample/31-graphql-federation-code-first/posts-application)，在模式优先模式下在[这里](https://github.com/nestjs/nest/tree/master/sample/32-graphql-federation-schema-first/posts-application)。

#### 联合示例：网关（Gateway）

首先安装所需的依赖：

```bash
$ npm install --save @apollo/gateway
```

网关需要指定一个端点列表，它将自动发现相应的模式。因此，网关服务的实现对于代码优先和模式优先方法都将保持不变。

```typescript
import { IntrospectAndCompose } from '@apollo/gateway';
import { ApolloGatewayDriver, ApolloGatewayDriverConfig } from '@nestjs/apollo';
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloGatewayDriverConfig>({
      driver: ApolloGatewayDriver,
      server: {
        // ... Apollo 服务器选项
        cors: true,
      },
      gateway: {
        supergraphSdl: new IntrospectAndCompose({
          subgraphs: [
            { name: 'users', url: 'http://user-service/graphql' },
            { name: 'posts', url: 'http://post-service/graphql' },
          ],
        }),
      },
    }),
  ],
})
export class AppModule {}
```

在代码优先模式下，可用的工作示例在[这里](https://github.com/nestjs/nest/tree/master/sample/31-graphql-federation-code-first/gateway)，在模式优先模式下在[这里](https://github.com/nestjs/nest/tree/master/sample/32-graphql-federation-schema-first/gateway)。

#### 使用 Mercurius 进行 Federation

首先安装所需的依赖：

```bash
$ npm install --save @apollo/subgraph @nestjs/mercurius
```

> info **注意** 构建子图模式（`buildSubgraphSchema`、`printSubgraphSchema` 函数）需要 `@apollo/subgraph` 包。

#### 模式优先（Schema first）

"用户服务"提供了一个简单的模式。注意 `@key` 指令：它指示 Mercurius 查询规划器，如果指定了 `id`，则可以获取特定的 `User` 实例。另外，请注意我们 `extend` 了 `Query` 类型。

```graphql
type User @key(fields: "id") {
  id: ID!
  name: String!
}

extend type Query {
  getUser(id: ID!): User
}
```

解析器提供了一个名为 `resolveReference()` 的额外方法。每当相关资源需要 User 实例时，Mercurius Gateway 就会触发该方法。我们稍后将在帖子服务中看到一个例子。请注意，该方法必须使用 `@ResolveReference()` 装饰器进行注解。

```typescript
import { Args, Query, Resolver, ResolveReference } from '@nestjs/graphql';
import { UsersService } from './users.service';

@Resolver('User')
export class UsersResolver {
  constructor(private usersService: UsersService) {}

  @Query()
  getUser(@Args('id') id: string) {
    return this.usersService.findById(id);
  }

  @ResolveReference()
  resolveReference(reference: { __typename: string; id: string }) {
    return this.usersService.findById(reference.id);
  }
}
```

最后，我们通过注册 `GraphQLModule` 并在配置对象中传入 `MercuriusFederationDriver` 驱动来将所有内容连接起来：

```typescript
import {
  MercuriusFederationDriver,
  MercuriusFederationDriverConfig,
} from '@nestjs/mercurius';
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';
import { UsersResolver } from './users.resolver';

@Module({
  imports: [
    GraphQLModule.forRoot<MercuriusFederationDriverConfig>({
      driver: MercuriusFederationDriver,
      typePaths: ['**/*.graphql'],
      federationMetadata: true,
    }),
  ],
  providers: [UsersResolver],
})
export class AppModule {}
```

#### 代码优先（Code first）

首先向 `User` 实体添加一些额外的装饰器。

```ts
import { Directive, Field, ID, ObjectType } from '@nestjs/graphql';

@ObjectType()
@Directive('@key(fields: "id")')
export class User {
  @Field(() => ID)
  id: number;

  @Field()
  name: string;
}
```

解析器提供了一个名为 `resolveReference()` 的额外方法。每当相关资源需要 User 实例时，Mercurius Gateway 就会触发该方法。我们稍后将在帖子服务中看到一个例子。请注意，该方法必须使用 `@ResolveReference()` 装饰器进行注解。

```ts
import { Args, Query, Resolver, ResolveReference } from '@nestjs/graphql';
import { User } from './user.entity';
import { UsersService } from './users.service';

@Resolver(() => User)
export class UsersResolver {
  constructor(private usersService: UsersService) {}

  @Query(() => User)
  getUser(@Args('id') id: number): User {
    return this.usersService.findById(id);
  }

  @ResolveReference()
  resolveReference(reference: { __typename: string; id: number }): User {
    return this.usersService.findById(reference.id);
  }
}
```

最后，我们通过注册 `GraphQLModule` 并在配置对象中传入 `MercuriusFederationDriver` 驱动来将所有内容连接起来：

```typescript
import {
  MercuriusFederationDriver,
  MercuriusFederationDriverConfig,
} from '@nestjs/mercurius';
import { Module } from '@nestjs/common';
import { UsersResolver } from './users.resolver';
import { UsersService } from './users.service'; // 此示例中未包含

@Module({
  imports: [
    GraphQLModule.forRoot<MercuriusFederationDriverConfig>({
      driver: MercuriusFederationDriver,
      autoSchemaFile: true,
      federationMetadata: true,
    }),
  ],
  providers: [UsersResolver, UsersService],
})
export class AppModule {}
```

#### 联合示例：帖子（Posts）

帖子服务旨在通过 `getPosts` 查询提供聚合的帖子，同时也使用 `user.posts` 字段扩展我们的 `User` 类型。

#### 模式优先（Schema first）

"帖子服务"在其模式中通过使用 `extend` 关键字引用 `User` 类型。它还在 `User` 类型上声明了一个额外的属性（`posts`）。注意用于匹配 User 实例的 `@key` 指令，以及指示 `id` 字段在其他地方管理的 `@external` 指令。

```graphql
type Post @key(fields: "id") {
  id: ID!
  title: String!
  body: String!
  user: User
}

extend type User @key(fields: "id") {
  id: ID! @external
  posts: [Post]
}

extend type Query {
  getPosts: [Post]
}
```

在以下示例中，`PostsResolver` 提供了 `getUser()` 方法，该方法返回一个包含 `__typename` 和您的应用程序可能需要解析引用的其他属性的引用，在这个例子中是 `id`。GraphQL Gateway 使用 `__typename` 来定位负责 User 类型的微服务并检索相应的实例。执行 `resolveReference()` 方法时，将请求上述的"用户服务"。

```typescript
import { Query, Resolver, Parent, ResolveField } from '@nestjs/graphql';
import { PostsService } from './posts.service';
import { Post } from './posts.interfaces';

@Resolver('Post')
export class PostsResolver {
  constructor(private postsService: PostsService) {}

  @Query('getPosts')
  getPosts() {
    return this.postsService.findAll();
  }

  @ResolveField('user')
  getUser(@Parent() post: Post) {
    return { __typename: 'User', id: post.userId };
  }
}
```

最后，我们必须注册 `GraphQLModule`，类似于我们在"用户服务"部分所做的。

```typescript
import {
  MercuriusFederationDriver,
  MercuriusFederationDriverConfig,
} from '@nestjs/mercurius';
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';
import { PostsResolver } from './posts.resolver';

@Module({
  imports: [
    GraphQLModule.forRoot<MercuriusFederationDriverConfig>({
      driver: MercuriusFederationDriver,
      federationMetadata: true,
      typePaths: ['**/*.graphql'],
    }),
  ],
  providers: [PostsResolvers],
})
export class AppModule {}
```

#### 代码优先（Code first）

首先，我们必须声明一个代表 `User` 实体的类。尽管实体本身存在于另一个服务中，但我们将在这里使用它（扩展其定义）。注意 `@extends` 和 `@external` 指令。

```ts
import { Directive, ObjectType, Field, ID } from '@nestjs/graphql';
import { Post } from './post.entity';

@ObjectType()
@Directive('@extends')
@Directive('@key(fields: "id")')
export class User {
  @Field(() => ID)
  @Directive('@external')
  id: number;

  @Field(() => [Post])
  posts?: Post[];
}
```

现在让我们为 `User` 实体上的扩展创建相应的解析器，如下所示：

```ts
import { Parent, ResolveField, Resolver } from '@nestjs/graphql';
import { PostsService } from './posts.service';
import { Post } from './post.entity';
import { User } from './user.entity';

@Resolver(() => User)
export class UsersResolver {
  constructor(private readonly postsService: PostsService) {}

  @ResolveField(() => [Post])
  public posts(@Parent() user: User): Post[] {
    return this.postsService.forAuthor(user.id);
  }
}
```

我们还必须定义 `Post` 实体类：

```ts
import { Directive, Field, ID, Int, ObjectType } from '@nestjs/graphql';
import { User } from './user.entity';

@ObjectType()
@Directive('@key(fields: "id")')
export class Post {
  @Field(() => ID)
  id: number;

  @Field()
  title: string;

  @Field(() => Int)
  authorId: number;

  @Field(() => User)
  user?: User;
}
```

及其解析器：

```ts
import { Query, Args, ResolveField, Resolver, Parent } from '@nestjs/graphql';
import { PostsService } from './posts.service';
import { Post } from './post.entity';
import { User } from './user.entity';

@Resolver(() => Post)
export class PostsResolver {
  constructor(private readonly postsService: PostsService) {}

  @Query(() => Post)
  findPost(@Args('id') id: number): Post {
    return this.postsService.findOne(id);
  }

  @Query(() => [Post])
  getPosts(): Post[] {
    return this.postsService.all();
  }

  @ResolveField(() => User)
  user(@Parent() post: Post): any {
    return { __typename: 'User', id: post.authorId };
  }
}
```

最后，在一个模块中将所有内容连接起来。注意模式构建选项，我们在其中指定 `User` 是一个孤立（外部）类型。

```ts
import {
  MercuriusFederationDriver,
  MercuriusFederationDriverConfig,
} from '@nestjs/mercurius';
import { Module } from '@nestjs/common';
import { User } from './user.entity';
import { PostsResolvers } from './posts.resolvers';
import { UsersResolvers } from './users.resolvers';
import { PostsService } from './posts.service'; // 此示例中未包含

@Module({
  imports: [
    GraphQLModule.forRoot<MercuriusFederationDriverConfig>({
      driver: MercuriusFederationDriver,
      autoSchemaFile: true,
      federationMetadata: true,
      buildSchemaOptions: {
        orphanedTypes: [User],
      },
    }),
  ],
  providers: [PostsResolver, UsersResolver, PostsService],
})
export class AppModule {}
```

#### 联合示例：网关（Gateway）

网关需要指定一个端点列表，它将自动发现相应的模式。因此，网关服务的实现对于代码优先和模式优先方法都将保持不变。

```typescript
import {
  MercuriusGatewayDriver,
  MercuriusGatewayDriverConfig,
} from '@nestjs/mercurius';
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';

@Module({
  imports: [
    GraphQLModule.forRoot<MercuriusGatewayDriverConfig>({
      driver: MercuriusGatewayDriver,
      gateway: {
        services: [
          { name: 'users', url: 'http://user-service/graphql' },
          { name: 'posts', url: 'http://post-service/graphql' },
        ],
      },
    }),
  ],
})
export class AppModule {}
```

### Federation 2

引用 [Apollo 文档](https://www.apollographql.com/docs/federation/federation-2/new-in-federation-2) 的说法，Federation 2 从原始 Apollo Federation（在本文档中称为 Federation 1）改进了开发者体验，并且与大多数原始超级图（supergraph）向后兼容。

> warning **警告** Mercurius 不完全支持 Federation 2。您可以在[这里](https://www.apollographql.com/docs/federation/supported-subgraphs#javascript--typescript)查看支持 Federation 2 的库列表。

在以下部分中，我们将把前面的示例升级到 Federation 2。

#### 联合示例：用户（Users）

Federation 2 的一个变化是实体没有起源子图，因此我们不再需要扩展 `Query`。更多详情请参考 Apollo Federation 2 文档中的[实体主题](https://www.apollographql.com/docs/federation/federation-2/new-in-federation-2#entities)。

#### 模式优先（Schema first）

我们可以简单地从模式中移除 `extend` 关键字。

```graphql
type User @key(fields: "id") {
  id: ID!
  name: String!
}

type Query {
  getUser(id: ID!): User
}
```

#### 代码优先（Code first）

要使用 Federation 2，我们需要在 `autoSchemaFile` 选项中指定 federation 版本。

```ts
import {
  ApolloFederationDriver,
  ApolloFederationDriverConfig,
} from '@nestjs/apollo';
import { Module } from '@nestjs/common';
import { UsersResolver } from './users.resolver';
import { UsersService } from './users.service'; // 此示例中未包含

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloFederationDriverConfig>({
      driver: ApolloFederationDriver,
      autoSchemaFile: {
        federation: 2,
      },
    }),
  ],
  providers: [UsersResolver, UsersService],
})
export class AppModule {}
```

#### 联合示例：帖子（Posts）

出于与上述相同的原因，我们不再需要扩展 `User` 和 `Query`。

#### 模式优先（Schema first）

我们可以简单地从模式中移除 `extend` 和 `external` 指令。

```graphql
type Post @key(fields: "id") {
  id: ID!
  title: String!
  body: String!
  user: User
}

type User @key(fields: "id") {
  id: ID!
  posts: [Post]
}

type Query {
  getPosts: [Post]
}
```

#### 代码优先（Code first）

由于我们不再扩展 `User` 实体，我们可以简单地从 `User` 中移除 `extends` 和 `external` 指令。

```ts
import { Directive, ObjectType, Field, ID } from '@nestjs/graphql';
import { Post } from './post.entity';

@ObjectType()
@Directive('@key(fields: "id")')
export class User {
  @Field(() => ID)
  id: number;

  @Field(() => [Post])
  posts?: Post[];
}
```

同样，与用户服务类似，我们需要在 `GraphQLModule` 中指定使用 Federation 2。

```ts
import {
  ApolloFederationDriver,
  ApolloFederationDriverConfig,
} from '@nestjs/apollo';
import { Module } from '@nestjs/common';
import { User } from './user.entity';
import { PostsResolvers } from './posts.resolvers';
import { UsersResolvers } from './users.resolvers';
import { PostsService } from './posts.service'; // 此示例中未包含

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloFederationDriverConfig>({
      driver: ApolloFederationDriver,
      autoSchemaFile: {
        federation: 2,
      },
      buildSchemaOptions: {
        orphanedTypes: [User],
      },
    }),
  ],
  providers: [PostsResolver, UsersResolver, PostsService],
})
export class AppModule {}
```