### 共享模型

> warning **警告** 本章节仅适用于代码优先方法。

在项目后端使用 TypeScript 的最大优势之一，是能够通过一个共享的 TypeScript 包，在基于 TypeScript 的前端应用中复用相同的模型。

但存在一个问题：使用代码优先方法创建的模型大量使用了与 GraphQL 相关的装饰器。这些装饰器在前端中是不相关的，并且会对性能产生负面影响。

#### 使用模型垫片

为了解决这个问题，NestJS 提供了一个“垫片”，允许你通过 `webpack`（或类似的工具）配置，用无实际作用的代码替换原有的装饰器。
要使用这个垫片，需要在 `@nestjs/graphql` 包和垫片之间配置一个别名。

例如，在 webpack 中是这样配置的：

```typescript
resolve: { // 参见：https://webpack.js.org/configuration/resolve/
  alias: {
      "@nestjs/graphql": path.resolve(__dirname, "../node_modules/@nestjs/graphql/dist/extra/graphql-model-shim")
  }
}
```

> info **提示** [TypeORM](/techniques/database) 包有一个类似的垫片，可以在[这里](https://github.com/typeorm/typeorm/blob/master/extra/typeorm-model-shim.js)找到。