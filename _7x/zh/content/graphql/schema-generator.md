### 生成 SDL

> warning **警告** 本章仅适用于代码优先方式。

要手动生成 GraphQL SDL 模式（即不运行应用程序、连接数据库、挂载解析器等），请使用 `GraphQLSchemaBuilderModule`。

```typescript
async function generateSchema() {
  const app = await NestFactory.create(GraphQLSchemaBuilderModule);
  await app.init();

  const gqlSchemaFactory = app.get(GraphQLSchemaFactory);
  const schema = await gqlSchemaFactory.create([RecipesResolver]);
  console.log(printSchema(schema));
}
```

> info **提示** `GraphQLSchemaBuilderModule` 和 `GraphQLSchemaFactory` 从 `@nestjs/graphql` 包导入。`printSchema` 函数从 `graphql` 包导入。

#### 用法

`gqlSchemaFactory.create()` 方法接收一个解析器类引用数组。例如：

```typescript
const schema = await gqlSchemaFactory.create([
  RecipesResolver,
  AuthorsResolver,
  PostsResolvers,
]);
```

它还可以接收第二个可选参数，一个标量类数组：

```typescript
const schema = await gqlSchemaFactory.create(
  [RecipesResolver, AuthorsResolver, PostsResolvers],
  [DurationScalar, DateScalar],
);
```

最后，您可以传递一个选项对象：

```typescript
const schema = await gqlSchemaFactory.create([RecipesResolver], {
  skipCheck: true,
  orphanedTypes: [],
});
```

- `skipCheck`: 忽略模式验证；布尔值，默认为 `false`
- `orphanedTypes`: 未显式引用（不属于对象图）但需要生成的类列表。通常，如果一个类被声明但未在图中被引用，它会被忽略。该属性的值是一个类引用数组。