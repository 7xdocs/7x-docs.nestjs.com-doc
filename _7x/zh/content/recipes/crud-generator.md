### CRUD 生成器 (仅限 TypeScript)

在项目的整个生命周期中，当我们构建新功能时，通常需要向应用程序添加新的资源。这些资源通常需要多次重复的操作，而我们每次定义新资源时都必须重复这些操作。

#### 介绍

让我们想象一个真实场景，我们需要为两个实体（例如 **User** 和 **Product** 实体）暴露 CRUD 端点。
遵循最佳实践，对于每个实体，我们将不得不执行以下几个操作：

- 生成一个模块 (`nest g mo`) 以保持代码组织有序并建立清晰的边界（将相关组件分组）
- 生成一个控制器 (`nest g co`) 来定义 CRUD 路由（或者对于 GraphQL 应用，定义查询和变更）
- 生成一个服务 (`nest g s`) 来实现和隔离业务逻辑
- 生成一个实体类/接口来表示资源的数据形态
- 生成数据传输对象（或者对于 GraphQL 应用，定义输入）来定义数据将如何通过网络发送

步骤非常多！

为了帮助加速这个重复的过程，[Nest CLI](/cli/overview) 提供了一个生成器（原理图），它能自动生成所有样板代码，帮助我们避免所有这些手动操作，并使开发体验更加简单。

> info **注意** 该原理图支持生成 **HTTP** 控制器、**微服务** 控制器、**GraphQL** 解析器（代码优先和架构优先都支持）以及 **WebSocket** 网关。

#### 生成新资源

要创建新资源，只需在项目的根目录下运行以下命令：

```shell
$ nest g resource
```

`nest g resource` 命令不仅生成所有的 NestJS 构建块（模块、服务、控制器类），还生成一个实体类、DTO 类以及测试 (`.spec`) 文件。

下面你可以看到生成的控制器文件（针对 REST API）：

```typescript
@Controller('users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Post()
  create(@Body() createUserDto: CreateUserDto) {
    return this.usersService.create(createUserDto);
  }

  @Get()
  findAll() {
    return this.usersService.findAll();
  }

  @Get(':id')
  findOne(@Param('id') id: string) {
    return this.usersService.findOne(+id);
  }

  @Patch(':id')
  update(@Param('id') id: string, @Body() updateUserDto: UpdateUserDto) {
    return this.usersService.update(+id, updateUserDto);
  }

  @Delete(':id')
  remove(@Param('id') id: string) {
    return this.usersService.remove(+id);
  }
}
```

同时，它会自动为所有 CRUD 端点创建占位符（REST API 的路由，GraphQL 的查询和变更，微服务和 WebSocket 网关的消息订阅）——所有这些都无需你动一根手指。

> warning **注意** 生成的服务类**不**与任何特定的 **ORM（或数据源）** 绑定。这使得生成器足够通用，以满足任何项目的需求。默认情况下，所有方法都将包含占位符，允许你填充特定于项目的数据源。

同样，如果你想为 GraphQL 应用程序生成解析器，只需选择 `GraphQL (code first)`（或 `GraphQL (schema first)`）作为你的传输层。

在这种情况下，NestJS 将生成一个解析器类，而不是 REST API 控制器：

```shell
$ nest g resource users

> ? What transport layer do you use? GraphQL (code first)
> ? Would you like to generate CRUD entry points? Yes
> CREATE src/users/users.module.ts (224 bytes)
> CREATE src/users/users.resolver.spec.ts (525 bytes)
> CREATE src/users/users.resolver.ts (1109 bytes)
> CREATE src/users/users.service.spec.ts (453 bytes)
> CREATE src/users/users.service.ts (625 bytes)
> CREATE src/users/dto/create-user.input.ts (195 bytes)
> CREATE src/users/dto/update-user.input.ts (281 bytes)
> CREATE src/users/entities/user.entity.ts (187 bytes)
> UPDATE src/app.module.ts (312 bytes)
```

> info **提示** 为了避免生成测试文件，你可以传递 `--no-spec` 标志，如下所示：`nest g resource users --no-spec`

我们可以在下面看到，不仅创建了所有样板变更和查询，而且所有内容都连接在一起。我们正在利用 `UsersService`、`User` 实体以及我们的 DTO。

```typescript
import { Resolver, Query, Mutation, Args, Int } from '@nestjs/graphql';
import { UsersService } from './users.service';
import { User } from './entities/user.entity';
import { CreateUserInput } from './dto/create-user.input';
import { UpdateUserInput } from './dto/update-user.input';

@Resolver(() => User)
export class UsersResolver {
  constructor(private readonly usersService: UsersService) {}

  @Mutation(() => User)
  createUser(@Args('createUserInput') createUserInput: CreateUserInput) {
    return this.usersService.create(createUserInput);
  }

  @Query(() => [User], { name: 'users' })
  findAll() {
    return this.usersService.findAll();
  }

  @Query(() => User, { name: 'user' })
  findOne(@Args('id', { type: () => Int }) id: number) {
    return this.usersService.findOne(id);
  }

  @Mutation(() => User)
  updateUser(@Args('updateUserInput') updateUserInput: UpdateUserInput) {
    return this.usersService.update(updateUserInput.id, updateUserInput);
  }

  @Mutation(() => User)
  removeUser(@Args('id', { type: () => Int }) id: number) {
    return this.usersService.remove(id);
  }
}
```