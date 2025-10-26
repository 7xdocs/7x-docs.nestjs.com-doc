### 映射类型

> warning **警告** 本章仅适用于代码优先（code first）方法。

在构建诸如 CRUD（创建/读取/更新/删除）等功能时，基于基础实体类型构造变体通常非常有用。Nest 提供了几个执行类型转换的工具函数，以使这项任务更加方便。

#### Partial（部分类型）

在构建输入验证类型（也称为数据传输对象或 DTO）时，通常需要在同一类型上构建 **create**（创建）和 **update**（更新）变体。例如，**create** 变体可能要求所有字段，而 **update** 变体可能将所有字段设为可选。

Nest 提供了 `PartialType()` 工具函数来简化此任务并减少样板代码。

`PartialType()` 函数返回一个类型（类），该类型将输入类型的所有属性设置为可选。例如，假设我们有一个 **create** 类型如下：

```typescript
@InputType()
class CreateUserInput {
  @Field()
  email: string;

  @Field()
  password: string;

  @Field()
  firstName: string;
}
```

默认情况下，所有这些字段都是必需的。要创建一个具有相同字段但每个字段都是可选的类型，请使用 `PartialType()` 并传入类引用（`CreateUserInput`）作为参数：

```typescript
@InputType()
export class UpdateUserInput extends PartialType(CreateUserInput) {}
```

> info **提示** `PartialType()` 函数从 `@nestjs/graphql` 包中导入。

`PartialType()` 函数接受一个可选的第二个参数，该参数是对装饰器工厂的引用。此参数可用于更改应用于结果（子）类的装饰器函数。如果未指定，子类将有效地使用与**父**类（第一个参数中引用的类）相同的装饰器。在上面的示例中，我们扩展了使用 `@InputType()` 装饰器注释的 `CreateUserInput`。由于我们希望 `UpdateUserInput` 也被视为使用了 `@InputType()` 装饰器，我们不需要将 `InputType` 作为第二个参数传递。如果父类和子类类型不同（例如，父类使用 `@ObjectType` 装饰），我们将把 `InputType` 作为第二个参数传递。例如：

```typescript
@InputType()
export class UpdateUserInput extends PartialType(User, InputType) {}
```

#### Pick（选择类型）

`PickType()` 函数通过从输入类型中选择一组属性来构造一个新类型（类）。例如，假设我们从一个如下类型开始：

```typescript
@InputType()
class CreateUserInput {
  @Field()
  email: string;

  @Field()
  password: string;

  @Field()
  firstName: string;
}
```

我们可以使用 `PickType()` 工具函数从这个类中选择一组属性：

```typescript
@InputType()
export class UpdateEmailInput extends PickType(CreateUserInput, [
  'email',
] as const) {}
```

> info **提示** `PickType()` 函数从 `@nestjs/graphql` 包中导入。

#### Omit（忽略类型）

`OmitType()` 函数通过从输入类型中选择所有属性，然后移除一组特定的键来构造一个类型。例如，假设我们从一个如下类型开始：

```typescript
@InputType()
class CreateUserInput {
  @Field()
  email: string;

  @Field()
  password: string;

  @Field()
  firstName: string;
}
```

我们可以生成一个派生类型，该类型拥有**除** `email` 之外的所有属性，如下所示。在此构造中，`OmitType` 的第二个参数是属性名称的数组。

```typescript
@InputType()
export class UpdateUserInput extends OmitType(CreateUserInput, [
  'email',
] as const) {}
```

> info **提示** `OmitType()` 函数从 `@nestjs/graphql` 包中导入。

#### Intersection（交叉类型）

`IntersectionType()` 函数将两个类型组合成一个新类型（类）。例如，假设我们有两个如下类型：

```typescript
@InputType()
class CreateUserInput {
  @Field()
  email: string;

  @Field()
  password: string;
}

@ObjectType()
export class AdditionalUserInfo {
  @Field()
  firstName: string;

  @Field()
  lastName: string;
}
```

我们可以生成一个新类型，它组合了两种类型中的所有属性。

```typescript
@InputType()
export class UpdateUserInput extends IntersectionType(
  CreateUserInput,
  AdditionalUserInfo,
) {}
```

> info **提示** `IntersectionType()` 函数从 `@nestjs/graphql` 包中导入。

#### 组合

类型映射工具函数是可组合的。例如，以下代码将生成一个类型（类），该类型拥有 `CreateUserInput` 类型除 `email` 外的所有属性，并且这些属性将被设置为可选：

```typescript
@InputType()
export class UpdateUserInput extends PartialType(
  OmitType(CreateUserInput, ['email'] as const),
) {}
```