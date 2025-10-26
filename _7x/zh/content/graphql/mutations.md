### Mutations

大多数关于 GraphQL 的讨论都集中在数据获取上，但任何完整的数据平台都需要一种修改服务端数据的方法。在 REST 中，任何请求都可能对服务器产生副作用，但最佳实践建议我们不应该在 GET 请求中修改数据。GraphQL 也是类似的——从技术上讲，任何查询都可以被实现为引起数据写入。然而，与 REST 一样，建议遵循以下约定：任何引起写入的操作都应该通过 mutation 显式发送（阅读更多[这里](https://graphql.org/learn/queries/#mutations)）。

官方的 [Apollo](https://www.apollographql.com/docs/graphql-tools/generate-schema.html) 文档使用了一个 `upvotePost()` mutation 示例。该 mutation 实现了一个增加帖子 `votes` 属性值的方法。为了在 Nest 中创建一个等效的 mutation，我们将使用 `@Mutation()` 装饰器。

#### 代码优先

让我们在上一节使用的 `AuthorResolver` 中添加另一个方法（参见 [解析器](/graphql/resolvers)）。

```typescript
@Mutation(() => Post)
async upvotePost(@Args({ name: 'postId', type: () => Int }) postId: number) {
  return this.postsService.upvoteById({ id: postId });
}
```

> info **提示** 所有的装饰器（例如 `@Resolver`、`@ResolveField`、`@Args` 等）都是从 `@nestjs/graphql` 包中导出的。

这将导致在 SDL 中生成以下 GraphQL 模式部分：

```graphql
type Mutation {
  upvotePost(postId: Int!): Post
}
```

`upvotePost()` 方法接受 `postId`（`Int`）作为参数，并返回一个更新后的 `Post` 实体。由于在[解析器](/graphql/resolvers)部分解释的原因，我们必须显式设置期望的类型。

如果 mutation 需要接受一个对象作为参数，我们可以创建一个**输入类型**。输入类型是一种特殊类型的对象类型，可以作为参数传递（阅读更多[这里](https://graphql.org/learn/schema/#input-types)）。要声明一个输入类型，请使用 `@InputType()` 装饰器。

```typescript
import { InputType, Field } from '@nestjs/graphql';

@InputType()
export class UpvotePostInput {
  @Field()
  postId: number;
}
```

> info **提示** `@InputType()` 装饰器接受一个选项对象作为参数，因此您可以，例如，指定输入类型的描述。请注意，由于 TypeScript 的元数据反射系统的限制，您必须使用 `@Field` 装饰器手动指示类型，或者使用 [CLI 插件](/graphql/cli-plugin)。

然后我们可以在解析器类中使用此类型：

```typescript
@Mutation(() => Post)
async upvotePost(
  @Args('upvotePostData') upvotePostData: UpvotePostInput,
) {}
```

#### 模式优先

让我们扩展上一节中使用的 `AuthorResolver`（参见 [解析器](/graphql/resolvers)）。

```typescript
@Mutation()
async upvotePost(@Args('postId') postId: number) {
  return this.postsService.upvoteById({ id: postId });
}
```

请注意，我们假设上述业务逻辑已移至 `PostsService`（查询帖子并增加其 `votes` 属性）。`PostsService` 类内部的逻辑可以根据需要简单或复杂。这个示例的主要目的是展示解析器如何与其他提供者交互。

最后一步是将我们的 mutation 添加到现有的类型定义中。

```graphql
type Author {
  id: Int!
  firstName: String
  lastName: String
  posts: [Post]
}

type Post {
  id: Int!
  title: String
  votes: Int
}

type Query {
  author(id: Int!): Author
}

type Mutation {
  upvotePost(postId: Int!): Post
}
```

`upvotePost(postId: Int!): Post` mutation 现在可以作为我们应用程序 GraphQL API 的一部分被调用。