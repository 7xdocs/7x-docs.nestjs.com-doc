### 概述

[Nest CLI](https://github.com/nestjs/nest-cli) 是一个命令行界面工具，可帮助您初始化、开发和维护 Nest 应用程序。它通过多种方式提供协助，包括搭建项目脚手架、在开发模式下提供服务，以及为生产分发构建和打包应用程序。它体现了最佳实践的架构模式，以鼓励构建结构良好的应用程序。

#### 安装

**注意**：在本指南中，我们描述了使用 [npm](https://docs.npmjs.com/downloading-and-installing-node-js-and-npm) 来安装包，包括 Nest CLI。您也可以自行决定使用其他包管理器。使用 npm 时，您有几个选项可以管理操作系统命令行如何解析 `nest` CLI 二进制文件的位置。这里，我们描述了使用 `-g` 选项全局安装 `nest` 二进制文件。这提供了一定的便利性，也是我们在整个文档中假设的方法。请注意，全局安装 **任何** `npm` 包后，确保它们运行正确版本的责任就落在了用户身上。这也意味着，如果您有不同的项目，每个项目都将运行 **相同** 版本的 CLI。一个合理的替代方案是使用内置于 `npm` cli 中的 [npx](https://github.com/npm/cli/blob/latest/docs/lib/content/commands/npx.md) 程序（或其他包管理器的类似功能），以确保您运行的是 Nest CLI 的 **受管理版本**。我们建议您查阅 [npx 文档](https://github.com/npm/cli/blob/latest/docs/lib/content/commands/npx.md) 和/或咨询您的 DevOps 支持人员以获取更多信息。

使用 `npm install -g` 命令全局安装 CLI（有关全局安装的详细信息，请参见上面的**注意**部分）。

```bash
$ npm install -g @nestjs/cli
```

> info **提示** 或者，您可以使用命令 `npx @nestjs/cli@latest` 而无需全局安装 CLI。

#### 基本工作流

安装后，您可以通过 `nest` 可执行文件直接从操作系统命令行调用 CLI 命令。通过输入以下命令查看可用的 `nest` 命令：

```bash
$ nest --help
```

使用以下结构获取单个命令的帮助。将下面示例中的 `generate` 替换为任何命令，如 `new`、`add` 等，以获取该命令的详细帮助：

```bash
$ nest generate --help
```

要在开发模式下创建、构建和运行一个新的基础 Nest 项目，请进入应作为新项目父级的文件夹，并运行以下命令：

```bash
$ nest new my-nest-project
$ cd my-nest-project
$ npm run start:dev
```

在您的浏览器中，打开 [http://localhost:3000](http://localhost:3000) 查看运行的新应用程序。当您更改任何源文件时，应用程序将自动重新编译和重新加载。

> info **提示** 我们建议使用 [SWC 构建器](/recipes/swc) 以获得更快的构建速度（比默认的 TypeScript 编译器性能高 10 倍）。

#### 项目结构

当您运行 `nest new` 时，Nest 会通过创建一个新文件夹并填充一组初始文件来生成一个样板应用程序结构。您可以继续在此默认结构中工作，并按照整个文档的描述添加新组件。我们将 `nest new` 生成的项目结构称为**标准模式**。Nest 还支持一种用于管理多个项目和库的替代结构，称为** monorepo 模式**。

除了围绕**构建**过程如何工作的几点特定考虑（本质上，monorepo 模式简化了有时可能由 monorepo 风格的项目结构引起的构建复杂性）以及内置的[库](/cli/libraries)支持之外，其余的 Nest 特性以及本文档同样适用于标准和 monorepo 模式的项目结构。事实上，您可以在将来任何时候轻松地从标准模式切换到 monorepo 模式，因此您可以在仍在学习 Nest 时安全地推迟这个决定。

您可以使用任一模式来管理多个项目。以下是差异的快速总结：

| 功能点                                                     | 标准模式                                                       | Monorepo 模式                                              |
| ---------------------------------------------------------- | ------------------------------------------------------------------ | ---------------------------------------------------------- |
| 多项目管理                                                 | 独立的文件系统结构                                                 | 单一文件系统结构                                           |
| `node_modules` 和 `package.json`                           | 独立的实例                                                         | 在 monorepo 内共享                                         |
| 默认编译器                                                 | `tsc`                                                              | webpack                                                    |
| 编译器设置                                                 | 单独指定                                                           | Monorepo 默认设置，可按项目覆盖                            |
| 配置文件，如 `eslint.config.mjs`, `.prettierrc` 等         | 单独指定                                                           | 在 monorepo 内共享                                         |
| `nest build` 和 `nest start` 命令                          | 目标自动默认为上下文中的（唯一）项目                               | 目标默认为 monorepo 中的**默认项目**                       |
| 库                                                         | 手动管理，通常通过 npm 打包                                        | 内置支持，包括路径管理和捆绑                               |

阅读关于[工作区](/cli/monorepo)和[库](/cli/libraries)的章节，以获取更详细的信息，帮助您决定哪种模式最适合您。

<app-banner-courses></app-banner-courses>

#### CLI 命令语法

所有 `nest` 命令都遵循相同的格式：

```bash
nest commandOrAlias requiredArg [optionalArg] [options]
```

例如：

```bash
$ nest new my-nest-project --dry-run
```

这里，`new` 是 _commandOrAlias_。`new` 命令有一个别名 `n`。`my-nest-project` 是 _requiredArg_。如果未在命令行提供 _requiredArg_，`nest` 将提示输入。此外，`--dry-run` 有一个等效的简写形式 `-d`。考虑到这一点，以下命令与上面的命令等效：

```bash
$ nest n my-nest-project -d
```

大多数命令和一些选项都有别名。尝试运行 `nest new --help` 来查看这些选项和别名，并确认您对上述结构的理解。

#### 命令概述

运行 `nest <command> --help` 查看以下任何命令的命令特定选项。

有关每个命令的详细说明，请参阅[用法](/cli/usages)。

| 命令       | 别名 | 描述                                                                                    |
| ---------- | ----- | ---------------------------------------------------------------------------------------------- |
| `new`      | `n`   | 搭建一个新的_标准模式_应用程序，包含运行所需的所有样板文件。          |
| `generate` | `g`   | 基于原理图生成和/或修改文件。                                          |
| `build`    |       | 将应用程序或工作区编译到输出文件夹。                                    |
| `start`    |       | 编译并运行应用程序（或工作区中的默认项目）。                          |
| `add`      |       | 导入一个已打包为 **nest 库**的库，并运行其安装原理图。 |
| `info`     | `i`   | 显示已安装的 nest 包的信息以及其他有用的系统信息。              |

#### 要求

Nest CLI 需要一个构建时包含[国际化支持](https://nodejs.org/api/intl.html) (ICU) 的 Node.js 二进制文件，例如来自 [Node.js 项目页面](https://nodejs.org/en/download) 的官方二进制文件。如果您遇到与 ICU 相关的错误，请检查您的二进制文件是否满足此要求。

```bash
node -p process.versions.icu
```

如果该命令打印出 `undefined`，则表示您的 Node.js 二进制文件没有国际化支持。