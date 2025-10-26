### CLI 命令参考

#### nest new

创建一个新的（标准模式）Nest 项目。

```bash
$ nest new <name> [options]
$ nest n <name> [options]
```

##### 描述

创建并初始化一个新的 Nest 项目。会提示选择包管理器。

- 创建一个具有给定 `<name>` 的文件夹
- 使用配置文件填充该文件夹
- 为源代码 (`/src`) 和端到端测试 (`/test`) 创建子文件夹
- 使用应用组件和测试的默认文件填充子文件夹

##### 参数

| 参数       | 描述                 |
| -------- | --------------------------- |
| `<name>` | 新项目的名称 |

##### 选项

| 选项                                | 描述                                                                                                                                                                                          |
| ------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `--dry-run`                           | 报告将要进行的更改，但不会更改文件系统。<br/> 别名: `-d`                                                                                                                                                                             |
| `--skip-git`                          | 跳过 git 仓库初始化。<br/> 别名: `-g`                                                                                                                                                 |
| `--skip-install`                      | 跳过包安装。<br/> 别名: `-s`                                                                                                                                                          |
| `--package-manager [package-manager]` | 指定包管理器。使用 `npm`、`yarn` 或 `pnpm`。包管理器必须全局安装。<br/> 别名: `-p`                                                                                  |
| `--language [language]`               | 指定编程语言 (`TS` 或 `JS`)。<br/> 别名: `-l`                                                                                                                                        |
| `--collection [collectionName]`       | 指定原理图集合。使用包含原理图的已安装 npm 包的包名称。<br/> 别名: `-c`                                                                                      |
| `--strict`                            | 在启用以下 TypeScript 编译器标志的情况下启动项目：`strictNullChecks`、`noImplicitAny`、`strictBindCallApply`、`forceConsistentCasingInFileNames`、`noFallthroughCasesInSwitch` |

#### nest generate

基于原理图生成和/或修改文件

```bash
$ nest generate <schematic> <name> [options]
$ nest g <schematic> <name> [options]
```

##### 参数

| 参数          | 描述                                                                                              |
| ------------- | -------------------------------------------------------------------------------------------------------- |
| `<schematic>` | 要生成的 `schematic` 或 `collection:schematic`。可用的原理图请参阅下表。 |
| `<name>`      | 生成的组件的名称。                                                                     |

##### 原理图

| 名称          | 别名 | 描述                                                                                                            |
| ------------- | ----- | ---------------------------------------------------------------------------------------------------------------------- |
| `app`         |       | 在 monorepo 中生成一个新的应用程序（如果是标准结构，则转换为 monorepo）。                    |
| `library`     | `lib` | 在 monorepo 中生成一个新的库（如果是标准结构，则转换为 monorepo）。                        |
| `class`       | `cl`  | 生成一个新的类。                                                                                                  |
| `controller`  | `co`  | 生成控制器声明。                                                                                     |
| `decorator`   | `d`   | 生成自定义装饰器。                                                                                           |
| `filter`      | `f`   | 生成过滤器声明。                                                                                         |
| `gateway`     | `ga`  | 生成网关声明。                                                                                        |
| `guard`       | `gu`  | 生成守卫声明。                                                                                          |
| `interface`   | `itf` | 生成一个接口。                                                                                                 |
| `interceptor` | `itc` | 生成拦截器声明。                                                                                   |
| `middleware`  | `mi`  | 生成中间件声明。                                                                                     |
| `module`      | `mo`  | 生成模块声明。                                                                                         |
| `pipe`        | `pi`  | 生成管道声明。                                                                                           |
| `provider`    | `pr`  | 生成提供者声明。                                                                                       |
| `resolver`    | `r`   | 生成解析器声明。                                                                                       |
| `resource`    | `res` | 生成一个新的 CRUD 资源。更多详情请参阅 [CRUD (resource) 生成器](/recipes/crud-generator)。 (仅限 TS) |
| `service`     | `s`   | 生成服务声明。                                                                                        |

##### 选项

| 选项                          | 描述                                                                                                     |
| ------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| `--dry-run`                     | 报告将要进行的更改，但不会更改文件系统。<br/> 别名: `-d`                        |
| `--project [project]`           | 元素应添加到的项目。<br/> 别名: `-p`                                                       |
| `--flat`                        | 不为元素生成文件夹。                                                                       |
| `--collection [collectionName]` | 指定原理图集合。使用包含原理图的已安装 npm 包的包名称。<br/> 别名: `-c` |
| `--spec`                        | 强制生成 spec 文件（默认）                                                                         |
| `--no-spec`                     | 禁用 spec 文件生成                                                                                   |

#### nest build

将应用程序或工作区编译到输出文件夹中。

此外，`build` 命令还负责：

- 通过 `tsconfig-paths` 映射路径（如果使用路径别名）
- 使用 OpenAPI 装饰器注解 DTO（如果启用了 `@nestjs/swagger` CLI 插件）
- 使用 GraphQL 装饰器注解 DTO（如果启用了 `@nestjs/graphql` CLI 插件）

```bash
$ nest build <name> [options]
```

##### 参数

| 参数       | 描述                       |
| -------- | --------------------------------- |
| `<name>` | 要构建的项目的名称。 |

##### 选项

| 选项                  | 描述                                                                                                                                                                                |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `--path [path]`         | `tsconfig` 文件的路径。 <br/>别名 `-p`                                                                                                                                                   |
| `--config [path]`       | `nest-cli` 配置文件的路径。 <br/>别名 `-c`                                                                                                                                     |
| `--watch`               | 在监视模式下运行（实时重新加载）。<br /> 如果使用 `tsc` 进行编译，可以输入 `rs` 来重新启动应用程序（当 `manualRestart` 选项设置为 `true` 时）。 <br/>别名 `-w` |
| `--builder [name]`      | 指定用于编译的构建器 (`tsc`, `swc`, 或 `webpack`)。 <br/>别名 `-b`                                                                                                   |
| `--webpack`             | 使用 webpack 进行编译（已弃用：请改用 `--builder webpack`）。                                                                                                                 |
| `--webpackPath`         | webpack 配置的路径。                                                                                                                                                             |
| `--tsc`                 | 强制使用 `tsc` 进行编译。                                                                                                                                                           |
| `--watchAssets`         | 监视非 TS 文件（资源，如 `.graphql` 等）。更多详情请参阅 [资源](cli/monorepo#assets)。                                                                                      |
| `--type-check`          | 启用类型检查（当使用 SWC 时）。                                                                                                                                                   |
| `--all`                 | 构建 monorepo 中的所有项目。                                                                                                                                                          |
| `--preserveWatchOutput` | 在监视模式下保留过时的控制台输出，而不是清屏。（仅限 `tsc` 监视模式）                                                                                         |

#### nest start

编译并运行应用程序（或工作区中的默认项目）。

```bash
$ nest start <name> [options]
```

##### 参数

| 参数       | 描述                     |
| -------- | ------------------------------- |
| `<name>` | 要运行的项目的名称。 |

##### 选项

| 选项                  | 描述                                                                                                                        |
| ----------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| `--path [path]`         | `tsconfig` 文件的路径。 <br/>别名 `-p`                                                                                           |
| `--config [path]`       | `nest-cli` 配置文件的路径。 <br/>别名 `-c`                                                                             |
| `--watch`               | 在监视模式下运行（实时重新加载） <br/>别名 `-w`                                                                                    |
| `--builder [name]`      | 指定用于编译的构建器 (`tsc`, `swc`, 或 `webpack`)。 <br/>别名 `-b`                                           |
| `--preserveWatchOutput` | 在监视模式下保留过时的控制台输出，而不是清屏。（仅限 `tsc` 监视模式）                                 |
| `--watchAssets`         | 在监视模式下运行（实时重新加载），监视非 TS 文件（资源）。更多详情请参阅 [资源](cli/monorepo#assets)。               |
| `--debug [hostport]`    | 在调试模式下运行（使用 --inspect 标志） <br/>别名 `-d`                                                                            |
| `--webpack`             | 使用 webpack 进行编译。（已弃用：请改用 `--builder webpack`）                                                         |
| `--webpackPath`         | webpack 配置的路径。                                                                                                     |
| `--tsc`                 | 强制使用 `tsc` 进行编译。                                                                                                   |
| `--exec [binary]`       | 要运行的二进制文件（默认: `node`）。 <br/>别名 `-e`                                                                                   |
| `--no-shell`            | 不在 shell 内生成子进程（参见 node 的 `child_process.spawn()` 方法文档）。                                      |
| `--env-file`            | 从相对于当前目录的文件加载环境变量，使其在 `process.env` 上对应用程序可用。 |
| `-- [key=value]`        | 可以通过 `process.argv` 引用的命令行参数。                                                                 |

#### nest add

导入一个已打包为 **nest 库** 的库，并运行其安装原理图。

```bash
$ nest add <name> [options]
```

##### 参数

| 参数       | 描述                        |
| -------- | ---------------------------------- |
| `<name>` | 要导入的库的名称。 |

#### nest info

显示有关已安装的 nest 包的信息以及其他有用的系统信息。例如：

```bash
$ nest info
```

```bash
 _   _             _      ___  _____  _____  _     _____
| \ | |           | |    |_  |/  ___|/  __ \| |   |_   _|
|  \| |  ___  ___ | |_     | |\ `--. | /  \/| |     | |
| . ` | / _ \/ __|| __|    | | `--. \| |    | |     | |
| |\  ||  __/\__ \| |_ /\__/ //\__/ /| \__/\| |_____| |_
\_| \_/ \___||___/ \__|\____/ \____/  \____/\_____/\___/

[System Information]
OS Version : macOS High Sierra
NodeJS Version : v20.18.0
[Nest Information]
microservices version : 10.0.0
websockets version : 10.0.0
testing version : 10.0.0
common version : 10.0.0
core version : 10.0.0
```
