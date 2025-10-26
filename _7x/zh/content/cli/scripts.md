### Nest CLI 与脚本

本节提供了关于 `nest` 命令如何与编译器和脚本交互的额外背景信息，以帮助 DevOps 人员管理开发环境。

Nest 应用程序是一个**标准**的 TypeScript 应用程序，需要先编译成 JavaScript 才能执行。有多种方式可以完成编译步骤，开发人员/团队可以自由选择最适合他们的方式。考虑到这一点，Nest 提供了一套开箱即用的工具，旨在实现以下目标：

- 提供一个标准的构建/执行流程，可通过命令行使用，并具有合理的默认值，"开箱即用"。
- 确保构建/执行流程是**开放**的，以便开发人员能够直接访问底层工具，使用其原生特性和选项进行自定义。
- 保持一个完全标准的 TypeScript/Node.js 框架，使得整个编译/部署/执行流水线可以由开发团队选择的任何外部工具来管理。

这一目标通过结合使用 `nest` 命令、本地安装的 TypeScript 编译器和 `package.json` 脚本来实现。下面我们将描述这些技术如何协同工作。这应有助于您理解构建/执行流程中每个步骤发生了什么，以及在必要时如何自定义该行为。

#### nest 二进制文件

`nest` 命令是一个操作系统级别的二进制文件（即从操作系统命令行运行）。该命令实际上涵盖了 3 个不同的领域，如下所述。我们建议您通过项目脚手架自动提供的 `package.json` 脚本来运行构建 (`nest build`) 和执行 (`nest start`) 子命令（如果您希望通过克隆代码库来开始，而不是运行 `nest new`，请参阅 [typescript starter](https://github.com/nestjs/typescript-starter)）。

#### 构建

`nest build` 是标准 `tsc` 编译器或 `swc` 编译器（针对[标准项目](https://docs.nestjs.com/cli/overview#project-structure)）或使用 `ts-loader` 的 webpack 打包器（针对[monorepos](https://docs.nestjs.com/cli/overview#project-structure)）的封装器。除了开箱即用地处理 `tsconfig-paths` 外，它没有添加任何其他编译特性或步骤。它存在的原因是大多数开发人员，尤其是在开始使用 Nest 时，不需要调整编译器选项（例如，`tsconfig.json` 文件），这有时可能很棘手。

有关更多详细信息，请参阅 [nest build](https://docs.nestjs.com/cli/usages#nest-build) 文档。

#### 执行

`nest start` 只是确保项目已构建（与 `nest build` 相同），然后以一种便携、简单的方式调用 `node` 命令来执行编译后的应用程序。与构建一样，您可以根据需要自由自定义此过程，可以使用 `nest start` 命令及其选项，也可以完全替换它。整个过程是一个标准的 TypeScript 应用程序构建和执行流水线，您可以自由地按此管理该过程。

有关更多详细信息，请参阅 [nest start](https://docs.nestjs.com/cli/usages#nest-start) 文档。

#### 生成

`nest generate` 命令，顾名思义，用于生成新的 Nest 项目或其中的组件。

#### Package 脚本

在操作系统命令行级别运行 `nest` 命令要求 `nest` 二进制文件已全局安装。这是 npm 的一个标准特性，不在 Nest 的直接控制范围内。这样做的一个后果是，全局安装的 `nest` 二进制文件**不**作为项目依赖项在 `package.json` 中管理。例如，两个不同的开发人员可能运行两个不同版本的 `nest` 二进制文件。对此的标准解决方案是使用 package 脚本，以便您可以将构建和执行步骤中使用的工具视为开发依赖项。

当您运行 `nest new` 或克隆 [typescript starter](https://github.com/nestjs/typescript-starter) 时，Nest 会使用诸如 `build` 和 `start` 之类的命令来填充新项目的 `package.json` 脚本。它还将底层编译器工具（例如 `typescript`）安装为**开发依赖项**。

您可以使用以下命令运行构建和执行脚本：

```bash
$ npm run build
```

和

```bash
$ npm run start
```

这些命令使用 npm 的脚本运行功能来执行 `nest build` 或 `nest start`，使用的是**本地安装**的 `nest` 二进制文件。通过使用这些内置的 package 脚本，您可以完全管理 Nest CLI 命令的依赖关系*。这意味着，通过遵循此**推荐**用法，您组织中的所有成员都可以确保运行相同版本的命令。

\*这适用于 `build` 和 `start` 命令。`nest new` 和 `nest generate` 命令不是构建/执行流水线的一部分，因此它们在不同的上下文中运行，并且没有内置的 `package.json` 脚本。

对于大多数开发人员/团队，建议利用 package 脚本来构建和执行他们的 Nest 项目。您可以通过这些脚本的选项（`--path`、`--webpack`、`--webpackPath`）完全自定义它们的行为，和/或根据需要自定义 `tsc` 或 webpack 编译器选项文件（例如，`tsconfig.json`）。您也可以自由运行完全自定义的构建过程来编译 TypeScript（甚至可以直接使用 `ts-node` 执行 TypeScript）。

#### 向后兼容性

由于 Nest 应用程序是纯 TypeScript 应用程序，旧版本的 Nest 构建/执行脚本将继续运行。您不需要升级它们。您可以在准备好时选择利用新的 `nest build` 和 `nest start` 命令，或者继续运行以前或自定义的脚本。

#### 迁移

虽然您不需要进行任何更改，但您可能希望迁移到使用新的 CLI 命令，而不是使用诸如 `tsc-watch` 或 `ts-node` 之类的工具。在这种情况下，只需全局和本地安装最新版本的 `@nestjs/cli`：

```bash
$ npm install -g @nestjs/cli
$ cd  /some/project/root/folder
$ npm install -D @nestjs/cli
```

然后，您可以将 `package.json` 中定义的 `scripts` 替换为以下内容：

```typescript
"build": "nest build",
"start": "nest start",
"start:dev": "nest start --watch",
"start:debug": "nest start --debug --watch",
```