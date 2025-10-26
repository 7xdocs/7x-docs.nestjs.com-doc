### 库

许多应用程序需要解决相同的通用问题，或在几种不同的上下文中复用模块化组件。Nest 有几种方法来解决这个问题，但每种方法都在不同层面上工作，以有助于满足不同架构和组织目标的方式来解决问题。

Nest [模块](/modules) 有助于提供一个执行上下文，使得可以在单个应用程序内共享组件。模块也可以通过 [npm](https://npmjs.com) 打包，以创建可在不同项目中安装的可复用库。对于分发可配置、可复用的库，供不同的、松散连接或无关的组织使用（例如，通过分发/安装第三方库），这可能是一种有效的方式。

对于在紧密组织的团体（例如，在公司/项目范围内）内共享代码，采用更轻量级的组件共享方法会很有用。Monorepo 应运而生，作为一种能够实现这一点的结构，并且在 monorepo 中，**库** 提供了一种简单、轻量级的代码共享方式。在 Nest monorepo 中，使用库能够轻松组装共享组件的应用程序。事实上，这鼓励了单体应用程序的分解和开发流程的转变，转而专注于构建和组合模块化组件。

#### Nest 库

Nest 库是一个与应用程序不同的 Nest 项目，因为它不能独立运行。库必须被导入到一个包含它的应用程序中，其代码才能执行。本节描述的内置库支持仅适用于 **monorepos**（标准模式项目可以通过使用 npm 包实现类似功能）。

例如，一个组织可能开发一个 `AuthModule`，通过实施管理所有内部应用程序的公司策略来管理身份验证。与其为每个应用程序单独构建该模块，或者通过 npm 物理打包代码并要求每个项目安装它，monorepo 可以将此模块定义为一个库。当以这种方式组织时，库模块的所有使用者都可以看到 `AuthModule` 在提交时的最新版本。这对于协调组件开发和组装，以及简化端到端测试具有显著的好处。

#### 创建库

任何适合复用的功能都是可以作为库来管理的候选者。决定什么应该成为库，什么应该成为应用程序的一部分，是一个架构设计决策。创建库不仅仅是简单地将代码从现有应用程序复制到新库中。当作为库打包时，库代码必须与应用程序解耦。这可能需要**更多**的前期时间，并迫使您做出一些在更紧密耦合的代码中可能不会面临的设计决策。但是，当该库能够用于在多个应用程序中实现更快速的应用程序组装时，这种额外的努力是值得的。

要开始创建库，请运行以下命令：

```bash
$ nest g library my-library
```

当您运行该命令时，`library` 原理图会提示您输入库的前缀（又名别名）：

```bash
What prefix would you like to use for the library (default: @app)?
```

这将在您的工作区中创建一个名为 `my-library` 的新项目。
一个库类型的项目，就像应用程序类型的项目一样，是通过原理图生成到一个指定名称的文件夹中的。库在 monorepo 根目录的 `libs` 文件夹下管理。Nest 在第一次创建库时会创建 `libs` 文件夹。

为库生成的文件与为应用程序生成的文件略有不同。以下是执行上述命令后 `libs` 文件夹的内容：

<div class="file-tree">
  <div class="item">libs</div>
  <div class="children">
    <div class="item">my-library</div>
    <div class="children">
      <div class="item">src</div>
      <div class="children">
        <div class="item">index.ts</div>
        <div class="item">my-library.module.ts</div>
        <div class="item">my-library.service.ts</div>
      </div>
      <div class="item">tsconfig.lib.json</div>
    </div>
  </div>
</div>

`nest-cli.json` 文件将在 `"projects"` 键下有一个新的库条目：

```javascript
...
{
    "my-library": {
      "type": "library",
      "root": "libs/my-library",
      "entryFile": "index",
      "sourceRoot": "libs/my-library/src",
      "compilerOptions": {
        "tsConfigPath": "libs/my-library/tsconfig.lib.json"
      }
}
...
```

库和应用程序在 `nest-cli.json` 元数据中有两个区别：

- `"type"` 属性设置为 `"library"` 而不是 `"application"`
- `"entryFile"` 属性设置为 `"index"` 而不是 `"main"`

这些差异指示构建过程以适当的方式处理库。例如，库通过 `index.js` 文件导出其功能。

与应用程序类型的项目一样，每个库都有自己的 `tsconfig.lib.json` 文件，该文件扩展了根（monorepo 范围内）的 `tsconfig.json` 文件。如果需要，您可以修改此文件以提供特定于库的编译器选项。

您可以使用 CLI 命令构建库：

```bash
$ nest build my-library
```

#### 使用库

有了自动生成的配置文件后，使用库就变得很简单。我们如何将 `MyLibraryService` 从 `my-library` 库导入到 `my-project` 应用程序中？

首先，请注意，使用库模块与使用任何其他 Nest 模块相同。Monorepo 所做的是以一种透明的方式管理路径，使得导入库和生成构建现在变得清晰明了。要使用 `MyLibraryService`，我们需要导入其声明模块。我们可以按如下方式修改 `my-project/src/app.module.ts` 来导入 `MyLibraryModule`。

```typescript
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { MyLibraryModule } from '@app/my-library';

@Module({
  imports: [MyLibraryModule],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

请注意，在上面我们在 ES 模块的 `import` 行中使用了 `@app` 的路径别名，这就是我们在上面的 `nest g library` 命令中提供的 `prefix`。在底层，Nest 通过 tsconfig 路径映射来处理这个问题。当添加一个库时，Nest 会更新全局（monorepo）`tsconfig.json` 文件的 `"paths"` 键，如下所示：

```javascript
"paths": {
    "@app/my-library": [
        "libs/my-library/src"
    ],
    "@app/my-library/*": [
        "libs/my-library/src/*"
    ]
}
```

因此，简而言之，monorepo 和库功能的结合使得将库模块包含到应用程序中变得简单直观。

同样的机制支持构建和部署由库组成的应用程序。一旦您导入了 `MyLibraryModule`，运行 `nest build` 会自动处理所有模块解析，并打包应用程序及其所有库依赖项，以便部署。Monorepo 的默认编译器是 **webpack**，因此生成的发布文件是一个单独的文件，将所有转译后的 JavaScript 文件打包成一个文件。您也可以按照<a href="https://docs.nestjs.com/cli/monorepo#global-compiler-options">此处</a>的描述切换到 `tsc`。