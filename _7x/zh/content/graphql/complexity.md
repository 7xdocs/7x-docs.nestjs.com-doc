### 复杂度

> warning **警告** 本章仅适用于代码优先（code first）方法。

查询复杂度（Query complexity）允许你定义某些字段的复杂程度，并限制查询的**最大复杂度**。其思路是通过一个简单的数字来定义每个字段的复杂度。通常的默认设置是为每个字段分配复杂度值 `1`。此外，GraphQL 查询的复杂度计算可以通过所谓的复杂度估算器（complexity estimators）来自定义。复杂度估算器是一个用于计算字段复杂度的简单函数。你可以向规则添加任意数量的复杂度估算器，它们会按顺序依次执行。第一个返回数字复杂度值的估算器将确定该字段的复杂度。

`@nestjs/graphql` 包与 [graphql-query-complexity](https://github.com/slicknode/graphql-query-complexity) 这类工具集成得非常好，它提供了一个基于成本分析的解决方案。通过这个库，你可以拒绝执行那些被认为成本过高（即过于复杂）的 GraphQL 查询。

#### 安装

首先，我们需要安装所需的依赖项才能开始使用。

```bash
$ npm install --save graphql-query-complexity
```

#### 快速开始

安装过程完成后，我们可以定义 `ComplexityPlugin` 类：

```typescript
import { GraphQLSchemaHost } from "@nestjs/graphql";
import { Plugin } from "@nestjs/apollo";
import {
  ApolloServerPlugin,
  GraphQLRequestListener,
} from 'apollo-server-plugin-base';
import { GraphQLError } from 'graphql';
import {
  fieldExtensionsEstimator,
  getComplexity,
  simpleEstimator,
} from 'graphql-query-complexity';

@Plugin()
export class ComplexityPlugin implements ApolloServerPlugin {
  constructor(private gqlSchemaHost: GraphQLSchemaHost) {}

  async requestDidStart(): Promise<GraphQLRequestListener> {
    const maxComplexity = 20;
    const { schema } = this.gqlSchemaHost;

    return {
      async didResolveOperation({ request, document }) {
        const complexity = getComplexity({
          schema,
          operationName: request.operationName,
          query: document,
          variables: request.variables,
          estimators: [
            fieldExtensionsEstimator(),
            simpleEstimator({ defaultComplexity: 1 }),
          ],
        });
        if (complexity > maxComplexity) {
          throw new GraphQLError(
            `Query is too complex: ${complexity}. Maximum allowed complexity: ${maxComplexity}`,
          );
        }
        console.log('Query Complexity:', complexity);
      },
    };
  }
}
```

出于演示目的，我们将最大允许复杂度指定为 `20`。在上面的示例中，我们使用了 2 个估算器：`simpleEstimator` 和 `fieldExtensionsEstimator`。

- `simpleEstimator`：该估算器为每个字段返回一个固定的复杂度值。
- `fieldExtensionsEstimator`：该字段扩展估算器会从你的 schema 的每个字段中提取复杂度值。

> info **提示** 请记得在任意模块的 `providers` 数组中添加这个类。

#### 字段级别复杂度

配置好此插件后，我们现在可以通过在 `@Field()` 装饰器中传入的选项对象内指定 `complexity` 属性来定义任何字段的复杂度，如下所示：

```typescript
@Field({ complexity: 3 })
title: string;
```

或者，你也可以定义估算函数：

```typescript
@Field({ complexity: (options: ComplexityEstimatorArgs) => ... })
title: string;
```

#### 查询/变更级别复杂度

此外，`@Query()` 和 `@Mutation()` 装饰器也可以指定 `complexity` 属性，如下所示：

```typescript
@Query({ complexity: (options: ComplexityEstimatorArgs) => options.args.count * options.childComplexity })
items(@Args('count') count: number) {
  return this.itemsService.getItems({ count });
}
```