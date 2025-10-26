### 常见错误

在使用 NestJS 进行开发时，随着对框架的学习，你可能会遇到各种各样的错误。

#### “无法解析依赖”错误

> info **提示** 查看 [NestJS 开发工具](/devtools/overview#investigating-the-cannot-resolve-dependency-error)，它可以帮助你轻松解决“无法解析依赖”错误。

最常见的错误消息可能是关于 Nest 无法解析提供者的依赖项。错误消息通常看起来像这样：

```bash
Nest can't resolve dependencies of the <provider> (?). Please make sure that the argument <unknown_token> at index [<index>] is available in the <module> context.

Potential solutions:
- Is <module> a valid NestJS module?
- If <unknown_token> is a provider, is it part of the current <module>?
- If <unknown_token> is exported from a separate @Module, is that module imported within <module>?
  @Module({
    imports: [ /* the Module containing <unknown_token> */ ]
  })
```

该错误最常见的原因是提供者不在模块的 `providers` 数组中。请确保提供者确实在 `providers` 数组中，并且遵循 [标准的 NestJS 提供者实践](/fundamentals/custom-providers#di-fundamentals)。

有一些常见的陷阱。其中一个是将提供者放在 `imports` 数组中。如果是这种情况，错误消息中应该出现模块名称的地方会显示提供者的名称。

如果在开发过程中遇到此错误，请查看错误消息中提到的模块，并查看其 `providers`。对于 `providers` 数组中的每个提供者，请确保模块可以访问其所有依赖项。通常，“功能模块”和“根模块”中会重复出现提供者，这意味着 Nest 会尝试实例化该提供者两次。更有可能的是，包含重复的 `<provider>` 的模块应该添加到“根模块”的 `imports` 数组中。

如果上面的 `<unknown_token>` 是 `dependency`，你可能存在循环文件导入。这与下面的 [循环依赖](/faq/common-errors#circular-dependency-error) 不同，因为它不是指提供者在构造函数中相互依赖，而只是意味着两个文件最终相互导入。一个常见的情况是，模块文件声明了一个令牌并导入了一个提供者，而该提供者又从模块文件中导入了令牌常量。如果你使用桶文件（barrel files），请确保你的桶导入不会最终导致这些循环导入。

如果上面的 `<unknown_token>` 是 `Object`，这意味着你在使用类型/接口注入时没有使用正确的提供者令牌。要解决此问题，请确保：

1. 你正在导入类引用，或使用带有 `@Inject()` 装饰器的自定义令牌。阅读 [自定义提供者页面](/fundamentals/custom-providers)，并且
2. 对于基于类的提供者，你正在导入具体的类，而不是仅通过 [`import type ...`](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-8.html#type-only-imports-and-export) 语法导入类型。

另外，请确保你没有最终将提供者注入到其自身，因为 NestJS 中不允许自我注入。当发生这种情况时，`<unknown_token>` 很可能等于 `<provider>`。

<app-banner-devtools></app-banner-devtools>

如果你处于 **monorepo  setup** 中，你可能会遇到与上面相同的错误，但涉及到名为 `ModuleRef` 的核心提供者作为 `<unknown_token>`：

```bash
Nest can't resolve dependencies of the <provider> (?).
Please make sure that the argument ModuleRef at index [<index>] is available in the <module> context.
...
```

这可能是因为你的项目最终加载了 `@nestjs/core` 包的两个 Node 模块，如下所示：

```text
.
├── package.json
├── apps
│   └── api
│       └── node_modules
│           └── @nestjs/bull
│               └── node_modules
│                   └── @nestjs/core
└── node_modules
    ├── (other packages)
    └── @nestjs/core
```

解决方案：

- 对于 **Yarn** 工作区，使用 [nohoist 功能](https://classic.yarnpkg.com/blog/2018/02/15/nohoist) 防止提升 `@nestjs/core` 包。
- 对于 **pnpm** 工作区，在其他模块中将 `@nestjs/core` 设置为 peerDependencies，并在导入该模块的应用程序的 package.json 中设置 `"dependenciesMeta": {{ '{' }}"other-module-name": {{ '{' }}"injected": true &#125;&#125;`。参见：[dependenciesmetainjected](https://pnpm.io/package_json#dependenciesmetainjected)

#### “循环依赖”错误

有时，你会发现在应用程序中难以避免 [循环依赖](https://docs.nestjs.com/fundamentals/circular-dependency)。你需要采取一些步骤来帮助 Nest 解决这些问题。由循环依赖引起的错误如下所示：

```bash
Nest cannot create the <module> instance.
The module at index [<index>] of the <module> "imports" array is undefined.

Potential causes:
- A circular dependency between modules. Use forwardRef() to avoid it. Read more: https://docs.nestjs.com/fundamentals/circular-dependency
- The module at index [<index>] is of type "undefined". Check your import statements and the type of the module.

Scope [<module_import_chain>]
# example chain AppModule -> FooModule
```

循环依赖可能源于提供者之间的相互依赖，或者 TypeScript 文件之间为了常量而相互依赖，例如从模块文件导出常量并在服务文件中导入它们。在后一种情况下，建议为常量创建一个单独的文件。在前一种情况下，请遵循循环依赖的指南，并确保模块**和**提供者都用 `forwardRef` 标记。

#### 调试依赖错误

除了手动验证依赖项是否正确外，从 Nest 8.1.0 开始，你可以将 `NEST_DEBUG` 环境变量设置为一个解析为真值的字符串，在 Nest 解析应用程序的所有依赖项时获取额外的日志信息。

<figure><img src="/assets/injector_logs.png" /></figure>

在上面的图片中，黄色字符串是正在注入的依赖项的宿主类，蓝色字符串是注入的依赖项的名称或其注入令牌，紫色字符串是正在搜索依赖项的模块。通过这一点，你通常可以追溯依赖项解析的过程，以及为什么会出现依赖注入问题。

#### “文件更改检测到”循环不断

使用 TypeScript 4.9 及以上版本的 Windows 用户可能会遇到此问题。
当你尝试在监视模式下运行应用程序时，例如 `npm run start:dev`，会看到无休止的循环日志消息：

```bash
XX:XX:XX AM - File change detected. Starting incremental compilation...
XX:XX:XX AM - Found 0 errors. Watching for file changes.
```

当你使用 NestJS CLI 在监视模式下启动应用程序时，它是通过调用 `tsc --watch` 来完成的，而从 TypeScript 4.9 版本开始，使用了一种 [新策略](https://devblogs.microsoft.com/typescript/announcing-typescript-4-9/#file-watching-now-uses-file-system-events) 来检测文件更改，这可能是导致此问题的原因。
为了解决这个问题，你需要在 tsconfig.json 文件中的 `"compilerOptions"` 选项之后添加一个设置，如下所示：

```bash
  "watchOptions": {
    "watchFile": "fixedPollingInterval"
  }
```

这告诉 TypeScript 使用轮询方法来检查文件更改，而不是文件系统事件（新的默认方法），后者在某些机器上可能会导致问题。
你可以在 [TypeScript 文档](https://www.typescriptlang.org/tsconfig#watch-watchDirectory) 中了解更多关于 `"watchFile"` 选项的信息。
```