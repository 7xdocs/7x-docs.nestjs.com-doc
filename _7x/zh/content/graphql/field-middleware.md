### 字段中间件

> warning **警告** 本章仅适用于代码优先方式。

字段中间件让你可以在一个字段被解析**之前或之后**运行任意代码。字段中间件可用于转换字段的结果、验证字段的参数，甚至检查字段级别的角色（例如，需要访问目标字段时，会执行中间件函数）。

你可以将多个中间件函数连接到一个字段。在这种情况下，它们将沿着链顺序调用，前一个中间件决定是否调用下一个。`middleware` 数组中中间件函数的顺序很重要。第一个解析器是“最外层”，因此它最先执行，也最后执行（类似于 `graphql-middleware` 包）。第二个解析器是“第二外层”，因此它第二个执行，倒数第二个结束。

#### 开始使用

让我们从一个简单的中间件开始，它将在字段值发送回客户端之前记录该值：

```typescript
import { FieldMiddleware, MiddlewareContext, NextFn } from '@nestjs/graphql';

const loggerMiddleware: FieldMiddleware = async (
  ctx: MiddlewareContext,
  next: NextFn,
) => {
  const value = await next();
  console.log(value);
  return value;
};
```

> info **提示** `MiddlewareContext` 是一个对象，由 GraphQL 解析器函数通常接收的相同参数组成（`{{ '{' }} source, args, context, info {{ '}' }}`），而 `NextFn` 是一个函数，让你可以执行堆栈中的下一个中间件（绑定到此字段）或实际的字段解析器。

> warning **警告** 字段中间件函数无法注入依赖项，也无法访问 Nest 的 DI 容器，因为它们被设计得非常轻量，并且不应执行任何可能耗时的操作（例如从数据库检索数据）。如果你需要调用外部服务/从数据源查询数据，你应该在绑定到根查询/变更处理器的守卫/拦截器中执行此操作，并将其分配给 `context` 对象，你可以从字段中间件内部（特别是从 `MiddlewareContext` 对象）访问该对象。

请注意，字段中间件必须匹配 `FieldMiddleware` 接口。在上面的示例中，我们首先运行 `next()` 函数（它执行实际的字段解析器并返回字段值），然后，我们将该值记录到终端。此外，从中间件函数返回的值会完全覆盖先前的值，由于我们不希望执行任何更改，因此我们直接返回原始值。

这样，我们就可以直接在 `@Field()` 装饰器中注册我们的中间件，如下所示：

```typescript
@ObjectType()
export class Recipe {
  @Field({ middleware: [loggerMiddleware] })
  title: string;
}
```

现在，每当我们请求 `Recipe` 对象类型的 `title` 字段时，原始字段的值将被记录到控制台。

> info **提示** 要了解如何通过使用[扩展](/graphql/extensions)功能实现字段级权限系统，请查看此[章节](/graphql/extensions#using-custom-metadata)。

> warning **警告** 字段中间件只能应用于 `ObjectType` 类。更多详情，请查看此[issue](https://github.com/nestjs/graphql/issues/2446)。

此外，如上所述，我们可以在中间件函数内部控制字段的值。为了演示，让我们将配方的标题（如果存在）转换为大写：

```typescript
const value = await next();
return value?.toUpperCase();
```

在这种情况下，每个标题在请求时都会自动转换为大写。

同样，你也可以将字段中间件绑定到自定义字段解析器（用 `@ResolveField()` 装饰器注释的方法），如下所示：

```typescript
@ResolveField(() => String, { middleware: [loggerMiddleware] })
title() {
  return 'Placeholder';
}
```

> warning **警告** 如果在字段解析器级别启用了增强器（[了解更多](/graphql/other-features#execute-enhancers-at-the-field-resolver-level)），字段中间件函数将在任何拦截器、守卫等**绑定到方法**的增强器之前运行（但在为查询或变更处理器注册的根级别增强器之后）。

#### 全局字段中间件

除了直接将中间件绑定到特定字段之外，你还可以全局注册一个或多个中间件函数。在这种情况下，它们将自动连接到你的对象类型的所有字段。

```typescript
GraphQLModule.forRoot({
  autoSchemaFile: 'schema.gql',
  buildSchemaOptions: {
    fieldMiddleware: [loggerMiddleware],
  },
}),
```

> info **提示** 全局注册的字段中间件函数将**先于**本地注册的中间件（那些直接绑定到特定字段的中间件）执行。