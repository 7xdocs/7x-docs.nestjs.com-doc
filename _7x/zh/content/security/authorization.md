### 授权

**授权**是指确定用户能够做什么的过程。例如，管理员用户被允许创建、编辑和删除帖子。非管理员用户仅被授权阅读帖子。

授权与身份验证是正交且独立的。然而，授权需要一个身份验证机制。

处理授权有许多不同的方法和策略。任何项目所采用的方法都取决于其特定的应用需求。本章介绍了几种授权方法，可以适应各种不同的需求。

#### 基础 RBAC 实现

基于角色的访问控制（**RBAC**）是一种策略中立的访问控制机制，围绕角色和权限定义。在本节中，我们将演示如何使用 Nest [守卫](/guards)实现一个非常基础的 RBAC 机制。

首先，我们创建一个代表系统中角色的 `Role` 枚举：

```typescript
@@filename(role.enum)
export enum Role {
  User = 'user',
  Admin = 'admin',
}
```

> info **提示** 在更复杂的系统中，您可以将角色存储在数据库中，或从外部身份验证提供者拉取。

有了这个之后，我们可以创建一个 `@Roles()` 装饰器。这个装饰器允许指定访问特定资源需要哪些角色。

```typescript
@@filename(roles.decorator)
import { SetMetadata } from '@nestjs/common';
import { Role } from '../enums/role.enum';

export const ROLES_KEY = 'roles';
export const Roles = (...roles: Role[]) => SetMetadata(ROLES_KEY, roles);
@@switch
import { SetMetadata } from '@nestjs/common';

export const ROLES_KEY = 'roles';
export const Roles = (...roles) => SetMetadata(ROLES_KEY, roles);
```

现在我们有了自定义的 `@Roles()` 装饰器，我们可以用它来装饰任何路由处理程序。

```typescript
@@filename(cats.controller)
@Post()
@Roles(Role.Admin)
create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
@@switch
@Post()
@Roles(Role.Admin)
@Bind(Body())
create(createCatDto) {
  this.catsService.create(createCatDto);
}
```

最后，我们创建一个 `RolesGuard` 类，它将分配给当前用户的角色与当前正在处理的路由所需的实际角色进行比较。为了访问路由的角色（自定义元数据），我们将使用 `Reflector` 辅助类，它由框架开箱即用地提供，并从 `@nestjs/core` 包中暴露。

```typescript
@@filename(roles.guard)
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Reflector } from '@nestjs/core';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<Role[]>(ROLES_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (!requiredRoles) {
      return true;
    }
    const { user } = context.switchToHttp().getRequest();
    return requiredRoles.some((role) => user.roles?.includes(role));
  }
}
@@switch
import { Injectable, Dependencies } from '@nestjs/common';
import { Reflector } from '@nestjs/core';

@Injectable()
@Dependencies(Reflector)
export class RolesGuard {
  constructor(reflector) {
    this.reflector = reflector;
  }

  canActivate(context) {
    const requiredRoles = this.reflector.getAllAndOverride(ROLES_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (!requiredRoles) {
      return true;
    }
    const { user } = context.switchToHttp().getRequest();
    return requiredRoles.some((role) => user.roles.includes(role));
  }
}
```

> info **提示** 有关在上下文相关的方式中使用 `Reflector` 的更多细节，请参阅执行上下文章节的[反射和元数据](/fundamentals/execution-context#reflection-and-metadata)部分。

> warning **注意** 这个示例被称为“**基础**”，因为我们只在路由处理程序级别检查角色的存在。在真实世界的应用中，您可能有一些涉及多个操作的端点/处理程序，其中每个操作都需要一组特定的权限。在这种情况下，您必须在业务逻辑中的某处提供一个检查角色的机制，这使得维护变得有些困难，因为没有集中的地方将权限与特定操作关联起来。

在这个例子中，我们假设 `request.user` 包含用户实例和允许的角色（在 `roles` 属性下）。在您的应用中，您可能会在自定义的**身份验证守卫**中建立这种关联 - 有关更多细节，请参阅[身份验证](/security/authentication)章节。

为了确保这个示例工作，您的 `User` 类必须如下所示：

```typescript
class User {
  // ...其他属性
  roles: Role[];
}
```

最后，确保注册 `RolesGuard`，例如在控制器级别或全局注册：

```typescript
providers: [
  {
    provide: APP_GUARD,
    useClass: RolesGuard,
  },
],
```

当权限不足的用户请求一个端点时，Nest 会自动返回以下响应：

```typescript
{
  "statusCode": 403,
  "message": "Forbidden resource",
  "error": "Forbidden"
}
```

> info **提示** 如果您想返回不同的错误响应，您应该抛出自己特定的异常，而不是返回一个布尔值。

<app-banner-courses-auth></app-banner-courses-auth>

#### 基于声明的授权

当创建一个身份时，它可能会被分配一个或多个由受信任方颁发的声明。声明是一个名称-值对，表示主体可以做什么，而不是主体是什么。

为了在 Nest 中实现基于声明的授权，您可以遵循我们在[RBAC](/security/authorization#basic-rbac-implementation)部分展示的相同步骤，但有一个显著区别：不是检查特定的角色，您应该比较**权限**。每个用户都将被分配一组权限。同样，每个资源/端点将定义访问它们需要哪些权限（例如，通过一个专门的 `@RequirePermissions()` 装饰器）。

```typescript
@@filename(cats.controller)
@Post()
@RequirePermissions(Permission.CREATE_CAT)
create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
@@switch
@Post()
@RequirePermissions(Permission.CREATE_CAT)
@Bind(Body())
create(createCatDto) {
  this.catsService.create(createCatDto);
}
```

> info **提示** 在上面的例子中，`Permission`（类似于我们在 RBAC 部分展示的 `Role`）是一个 TypeScript 枚举，包含系统中所有可用的权限。

#### 集成 CASL

[CASL](https://casl.js.org/) 是一个同构的授权库，它限制了给定客户端允许访问的资源。它被设计为可逐步采用，并且可以轻松地在简单的基于声明和完全功能的基于主题和属性的授权之间扩展。

首先，安装 `@casl/ability` 包：

```bash
$ npm i @casl/ability
```

> info **提示** 在这个例子中，我们选择了 CASL，但您可以根据自己的偏好和项目需求使用任何其他库，如 `accesscontrol` 或 `acl`。

安装完成后，为了说明 CASL 的机制，我们将定义两个实体类：`User` 和 `Article`。

```typescript
class User {
  id: number;
  isAdmin: boolean;
}
```

`User` 类由两个属性组成，`id` 是唯一的用户标识符，`isAdmin` 表示用户是否具有管理员权限。

```typescript
class Article {
  id: number;
  isPublished: boolean;
  authorId: number;
}
```

`Article` 类有三个属性，分别是 `id`、`isPublished` 和 `authorId`。`id` 是唯一的文章标识符，`isPublished` 表示文章是否已经发布，`authorId` 是撰写文章的用户 ID。

现在让我们回顾并细化这个示例的需求：

- 管理员可以管理（创建/读取/更新/删除）所有实体
- 用户对所有内容具有只读访问权限
- 用户可以更新自己的文章（`article.authorId === userId`）
- 已经发布的文章不能被删除（`article.isPublished === true`）

考虑到这些，我们可以开始创建一个 `Action` 枚举，代表用户可以对实体执行的所有可能操作：

```typescript
export enum Action {
  Manage = 'manage',
  Create = 'create',
  Read = 'read',
  Update = 'update',
  Delete = 'delete',
}
```

> warning **注意** `manage` 是 CASL 中的一个特殊关键字，代表“任何”操作。

为了封装 CASL 库，我们现在生成 `CaslModule` 和 `CaslAbilityFactory`。

```bash
$ nest g module casl
$ nest g class casl/casl-ability.factory
```

有了这些之后，我们可以在 `CaslAbilityFactory` 上定义 `createForUser()` 方法。这个方法将为给定用户创建 `Ability` 对象：

```typescript
type Subjects = InferSubjects<typeof Article | typeof User> | 'all';

export type AppAbility = Ability<[Action, Subjects]>;

@Injectable()
export class CaslAbilityFactory {
  createForUser(user: User) {
    const { can, cannot, build } = new AbilityBuilder<
      Ability<[Action, Subjects]>
    >(Ability as AbilityClass<AppAbility>);

    if (user.isAdmin) {
      can(Action.Manage, 'all'); // 对所有内容具有读写访问权限
    } else {
      can(Action.Read, 'all'); // 对所有内容具有只读访问权限
    }

    can(Action.Update, Article, { authorId: user.id });
    cannot(Action.Delete, Article, { isPublished: true });

    return build({
      // 详情请阅读：https://casl.js.org/v6/en/guide/subject-type-detection#use-classes-as-subject-types
      detectSubjectType: (item) =>
        item.constructor as ExtractSubjectType<Subjects>,
    });
  }
}
```

> warning **注意** `all` 是 CASL 中的一个特殊关键字，代表“任何主题”。

> info **提示** `Ability`、`AbilityBuilder`、`AbilityClass` 和 `ExtractSubjectType` 类是从 `@casl/ability` 包中导出的。

> info **提示** `detectSubjectType` 选项让 CASL 理解如何从对象中获取主题类型。有关更多信息，请阅读 [CASL 文档](https://casl.js.org/v6/en/guide/subject-type-detection#use-classes-as-subject-types) 了解详情。

在上面的例子中，我们使用 `AbilityBuilder` 类创建了 `Ability` 实例。如您所料，`can` 和 `cannot` 接受相同的参数但含义不同，`can` 允许对指定主题执行操作，而 `cannot` 禁止。两者都可以接受最多 4 个参数。要了解这些函数的更多信息，请访问官方 [CASL 文档](https://casl.js.org/v6/en/guide/intro)。

最后，确保将 `CaslAbilityFactory` 添加到 `CaslModule` 模块定义的 `providers` 和 `exports` 数组中：

```typescript
import { Module } from '@nestjs/common';
import { CaslAbilityFactory } from './casl-ability.factory';

@Module({
  providers: [CaslAbilityFactory],
  exports: [CaslAbilityFactory],
})
export class CaslModule {}
```

这样，我们就可以使用标准的构造函数注入将 `CaslAbilityFactory` 注入到任何类中，只要 `CaslModule` 在宿主上下文中被导入：

```typescript
constructor(private caslAbilityFactory: CaslAbilityFactory) {}
```

然后在类中如下使用它。

```typescript
const ability = this.caslAbilityFactory.createForUser(user);
if (ability.can(Action.Read, 'all')) {
  // "user" 具有对所有内容的读取权限
}
```

> info **提示** 有关 `Ability` 类的更多信息，请参阅官方 [CASL 文档](https://casl.js.org/v6/en/guide/intro)。

例如，假设我们有一个不是管理员的用户。在这种情况下，用户应该能够阅读文章，但禁止创建新文章或删除现有文章。

```typescript
const user = new User();
user.isAdmin = false;

const ability = this.caslAbilityFactory.createForUser(user);
ability.can(Action.Read, Article); // true
ability.can(Action.Delete, Article); // false
ability.can(Action.Create, Article); // false
```

> info **提示** 尽管 `Ability` 和 `AbilityBuilder` 类都提供了 `can` 和 `cannot` 方法，但它们的目的不同，并且接受的参数略有不同。

此外，正如我们在需求中指定的，用户应该能够更新自己的文章：

```typescript
const user = new User();
user.id = 1;

const article = new Article();
article.authorId = user.id;

const ability = this.caslAbilityFactory.createForUser(user);
ability.can(Action.Update, article); // true

article.authorId = 2;
ability.can(Action.Update, article); // false
```

如您所见，`Ability` 实例允许我们以相当可读的方式检查权限。同样，`AbilityBuilder` 允许我们以类似的方式定义权限（并指定各种条件）。要查找更多示例，请访问官方文档。

#### 高级：实现 `PoliciesGuard`

在本节中，我们将演示如何构建一个更为复杂的守卫，它检查用户是否满足可以在方法级别配置的特定**授权策略**（您也可以扩展它以遵循在类级别配置的策略）。在这个例子中，我们将使用 CASL 包仅用于说明目的，但使用这个库不是必需的。此外，我们将使用我们在上一节中创建的 `CaslAbilityFactory` 提供者。

首先，让我们详细说明需求。目标是提供一个机制，允许每个路由处理程序指定策略检查。我们将支持对象和函数（用于更简单的检查以及更喜欢函数式风格代码的人）。

让我们从定义策略处理程序的接口开始：

```typescript
import { AppAbility } from '../casl/casl-ability.factory';

interface IPolicyHandler {
  handle(ability: AppAbility): boolean;
}

type PolicyHandlerCallback = (ability: AppAbility) => boolean;

export type PolicyHandler = IPolicyHandler | PolicyHandlerCallback;
```

如上所述，我们提供了两种定义策略处理程序的可能方式：一个对象（实现 `IPolicyHandler` 接口的类的实例）和一个函数（满足 `PolicyHandlerCallback` 类型）。

有了这个之后，我们可以创建一个 `@CheckPolicies()` 装饰器。这个装饰器允许指定必须满足哪些策略才能访问特定资源。

```typescript
export const CHECK_POLICIES_KEY = 'check_policy';
export const CheckPolicies = (...handlers: PolicyHandler[]) =>
  SetMetadata(CHECK_POLICIES_KEY, handlers);
```

现在让我们创建一个 `PoliciesGuard`，它将提取并执行绑定到路由处理程序的所有策略处理程序。

```typescript
@Injectable()
export class PoliciesGuard implements CanActivate {
  constructor(
    private reflector: Reflector,
    private caslAbilityFactory: CaslAbilityFactory,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const policyHandlers =
      this.reflector.get<PolicyHandler[]>(
        CHECK_POLICIES_KEY,
        context.getHandler(),
      ) || [];

    const { user } = context.switchToHttp().getRequest();
    const ability = this.caslAbilityFactory.createForUser(user);

    return policyHandlers.every((handler) =>
      this.execPolicyHandler(handler, ability),
    );
  }

  private execPolicyHandler(handler: PolicyHandler, ability: AppAbility) {
    if (typeof handler === 'function') {
      return handler(ability);
    }
    return handler.handle(ability);
  }
}
```

> info **提示** 在这个例子中，我们假设 `request.user` 包含用户实例。在您的应用中，您可能会在自定义的**身份验证守卫**中建立这种关联 - 有关更多细节，请参阅[身份验证](/security/authentication)章节。

让我们分解这个例子。`policyHandlers` 是通过 `@CheckPolicies()` 装饰器分配给方法的处理程序数组。接下来，我们使用 `CaslAbilityFactory#create` 方法构造 `Ability` 对象，允许我们验证用户是否具有执行特定操作的足够权限。我们将这个对象传递给策略处理程序，它要么是一个函数，要么是一个实现 `IPolicyHandler` 的类的实例，暴露返回布尔值的 `handle()` 方法。最后，我们使用 `Array#every` 方法确保每个处理程序都返回 `true` 值。

最后，为了测试这个守卫，将其绑定到任何路由处理程序，并注册一个内联策略处理程序（函数式方法），如下所示：

```typescript
@Get()
@UseGuards(PoliciesGuard)
@CheckPolicies((ability: AppAbility) => ability.can(Action.Read, Article))
findAll() {
  return this.articlesService.findAll();
}
```

或者，我们可以定义一个实现 `IPolicyHandler` 接口的类：

```typescript
export class ReadArticlePolicyHandler implements IPolicyHandler {
  handle(ability: AppAbility) {
    return ability.can(Action.Read, Article);
  }
}
```

并如下使用它：

```typescript
@Get()
@UseGuards(PoliciesGuard)
@CheckPolicies(new ReadArticlePolicyHandler())
findAll() {
  return this.articlesService.findAll();
}
```

> warning **注意** 由于我们必须使用 `new` 关键字就地实例化策略处理程序，`ReadArticlePolicyHandler` 类不能使用依赖注入。这可以通过 `ModuleRef#get` 方法解决（阅读更多[这里](/fundamentals/module-ref)）。基本上，不是通过 `@CheckPolicies()` 装饰器注册函数和实例，您必须允许传递一个 `Type<IPolicyHandler>`。然后，在您的守卫内部，您可以使用类型引用检索一个实例：`moduleRef.get(YOUR_HANDLER_TYPE)` 或者甚至使用 `ModuleRef#create` 方法动态实例化它。