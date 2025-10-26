### 循环依赖

当两个类相互依赖时，就会出现循环依赖。例如，类 A 需要类 B，而类 B 也需要类 A。在 Nest 中，模块之间和提供者之间可能会出现循环依赖。

虽然应尽可能避免循环依赖，但并非总能做到。在这种情况下，Nest 提供了两种方式来解决提供者之间的循环依赖。在本章中，我们将介绍使用**前向引用**作为一种技术，以及使用 **ModuleRef** 类从 DI 容器中获取提供者实例作为另一种技术。

我们还将介绍如何解决模块之间的循环依赖。

> warning **警告** 当使用“桶文件”/index.ts 文件来分组导入时，也可能导致循环依赖。在涉及模块/提供者类时，应省略桶文件。例如，在导入与桶文件同一目录下的文件时，不应使用桶文件，即 `cats/cats.controller` 不应导入 `cats` 来导入 `cats/cats.service` 文件。更多详情请参阅 [此 GitHub issue](https://github.com/nestjs/nest/issues/1181#issuecomment-430197191)。

#### 前向引用

**前向引用**允许 Nest 使用 `forwardRef()` 工具函数引用尚未定义的类。例如，如果 `CatsService` 和 `CommonService` 相互依赖，关系的双方可以使用 `@Inject()` 和 `forwardRef()` 工具函数来解决循环依赖。否则 Nest 将无法实例化它们，因为所有必需的元数据都将不可用。以下是一个示例：

```typescript
@@filename(cats.service)
@Injectable()
export class CatsService {
  constructor(
    @Inject(forwardRef(() => CommonService))
    private commonService: CommonService,
  ) {}
}
@@switch
@Injectable()
@Dependencies(forwardRef(() => CommonService))
export class CatsService {
  constructor(commonService) {
    this.commonService = commonService;
  }
}
```

> info **提示** `forwardRef()` 函数是从 `@nestjs/common` 包中导入的。

这涵盖了关系的一侧。现在让我们对 `CommonService` 做同样的处理：

```typescript
@@filename(common.service)
@Injectable()
export class CommonService {
  constructor(
    @Inject(forwardRef(() => CatsService))
    private catsService: CatsService,
  ) {}
}
@@switch
@Injectable()
@Dependencies(forwardRef(() => CatsService))
export class CommonService {
  constructor(catsService) {
    this.catsService = catsService;
  }
}
```

> warning **警告** 实例化的顺序是不确定的。请确保您的代码不依赖于哪个构造函数先被调用。如果循环依赖的提供者使用了 `Scope.REQUEST`，可能会导致未定义的依赖关系。更多信息请参阅[此处](https://github.com/nestjs/nest/issues/5778)。

#### ModuleRef 类替代方案

使用 `forwardRef()` 的另一种方法是重构代码，并使用 `ModuleRef` 类在（原本是）循环关系的一侧获取提供者。了解更多关于 `ModuleRef` 工具类的信息，请参阅[这里](/fundamentals/module-ref)。

#### 模块前向引用

为了解决模块之间的循环依赖，需要在模块关联的两侧使用相同的 `forwardRef()` 工具函数。例如：

```typescript
@@filename(common.module)
@Module({
  imports: [forwardRef(() => CatsModule)],
})
export class CommonModule {}
```

这涵盖了关系的一侧。现在让我们对 `CatsModule` 做同样的处理：

```typescript
@@filename(cats.module)
@Module({
  imports: [forwardRef(() => CommonModule)],
})
export class CatsModule {}
```