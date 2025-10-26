### 引言

Nest (NestJS) 是一个用于构建高效、可扩展的 [Node.js](https://nodejs.org/) 服务端应用程序的框架。它使用渐进式 JavaScript，使用 [TypeScript](http://www.typescriptlang.org/) 构建并完全支持 TypeScript（但仍然允许开发者使用纯 JavaScript 编码），并结合了 OOP（面向对象编程）、FP（函数式编程）和 FRP（函数式响应式编程）的元素。

在底层，Nest 使用了健壮的 HTTP 服务器框架，如 [Express](https://expressjs.com/)（默认），并且可以配置为使用 [Fastify](https://github.com/fastify/fastify)！

Nest 在这些常见的 Node.js 框架（Express/Fastify）之上提供了一定程度的抽象，同时也直接向开发者暴露了它们的 API。这使开发者能够自由使用底层平台提供的众多第三方模块。

#### 设计哲学

近年来，得益于 Node.js，JavaScript 已成为前端和后端应用程序在网络领域的"通用语言"。这催生了许多出色的项目，如 [Angular](https://angular.dev/)、[React](https://github.com/facebook/react) 和 [Vue](https://github.com/vuejs/vue)，它们提高了开发者的生产力，使得能够创建快速、可测试且可扩展的前端应用程序。然而，虽然 Node（以及服务端 JavaScript）存在大量优秀的库、辅助程序和工具，但它们都没有有效地解决主要问题——**架构**。

Nest 提供了一个开箱即用的应用程序架构，允许开发者和团队创建高度可测试、可扩展、松散耦合且易于维护的应用程序。该架构深受 Angular 的启发。

#### 安装

要开始使用，您可以使用 [Nest CLI](/cli/overview) 搭建项目，或者[克隆一个入门项目](#alternatives)（两种方式会产生相同的结果）。

要使用 Nest CLI 搭建项目，请运行以下命令。这将创建一个新的项目目录，并用初始的核心 Nest 文件和支持模块填充该目录，为您的项目创建一个常规的基础结构。对于初次使用者，推荐使用 **Nest CLI** 创建新项目。我们将在[第一步](first-steps)中继续使用此方法。

```bash
$ npm i -g @nestjs/cli
$ nest new project-name
```

> info **提示** 要创建一个具有更严格功能集的新的 TypeScript 项目，请将 `--strict` 标志传递给 `nest new` 命令。

#### 备选方案

或者，使用 **Git** 安装 TypeScript 入门项目：

```bash
$ git clone https://github.com/nestjs/typescript-starter.git project
$ cd project
$ npm install
$ npm run start
```

> info **提示** 如果您想在没有 git 历史记录的情况下克隆仓库，可以使用 [degit](https://github.com/Rich-Harris/degit)。

打开浏览器并访问 [`http://localhost:3000/`](http://localhost:3000/)。

要安装 JavaScript 版本的入门项目，请在上述命令序列中使用 `javascript-starter.git`。

您也可以通过安装核心和支持包来从头开始一个新项目。请记住，您需要自行设置项目样板文件。至少，您需要这些依赖项：`@nestjs/core`、`@nestjs/common`、`rxjs` 和 `reflect-metadata`。查看这篇关于如何创建完整项目的简短文章：[5 steps to create a bare minimum NestJS app from scratch!](https://dev.to/micalevisk/5-steps-to-create-a-bare-minimum-nestjs-app-from-scratch-5c3b)。### 引言

Nest (NestJS) 是一个用于构建高效、可扩展的 [Node.js](https://nodejs.org/) 服务端应用程序的框架。它使用渐进式 JavaScript，使用 [TypeScript](http://www.typescriptlang.org/) 构建并完全支持 TypeScript（但仍然允许开发者使用纯 JavaScript 编码），并结合了 OOP（面向对象编程）、FP（函数式编程）和 FRP（函数式响应式编程）的元素。

在底层，Nest 使用了健壮的 HTTP 服务器框架，如 [Express](https://expressjs.com/)（默认），并且可以配置为使用 [Fastify](https://github.com/fastify/fastify)！

Nest 在这些常见的 Node.js 框架（Express/Fastify）之上提供了一定程度的抽象，同时也直接向开发者暴露了它们的 API。这使开发者能够自由使用底层平台提供的众多第三方模块。

#### 设计哲学

近年来，得益于 Node.js，JavaScript 已成为前端和后端应用程序在网络领域的"通用语言"。这催生了许多出色的项目，如 [Angular](https://angular.dev/)、[React](https://github.com/facebook/react) 和 [Vue](https://github.com/vuejs/vue)，它们提高了开发者的生产力，使得能够创建快速、可测试且可扩展的前端应用程序。然而，虽然 Node（以及服务端 JavaScript）存在大量优秀的库、辅助程序和工具，但它们都没有有效地解决主要问题——**架构**。

Nest 提供了一个开箱即用的应用程序架构，允许开发者和团队创建高度可测试、可扩展、松散耦合且易于维护的应用程序。该架构深受 Angular 的启发。

#### 安装

要开始使用，您可以使用 [Nest CLI](/cli/overview) 搭建项目，或者[克隆一个入门项目](#alternatives)（两种方式会产生相同的结果）。

要使用 Nest CLI 搭建项目，请运行以下命令。这将创建一个新的项目目录，并用初始的核心 Nest 文件和支持模块填充该目录，为您的项目创建一个常规的基础结构。对于初次使用者，推荐使用 **Nest CLI** 创建新项目。我们将在[第一步](first-steps)中继续使用此方法。

```bash
$ npm i -g @nestjs/cli
$ nest new project-name
```

> info **提示** 要创建一个具有更严格功能集的新的 TypeScript 项目，请将 `--strict` 标志传递给 `nest new` 命令。

#### 备选方案

或者，使用 **Git** 安装 TypeScript 入门项目：

```bash
$ git clone https://github.com/nestjs/typescript-starter.git project
$ cd project
$ npm install
$ npm run start
```

> info **提示** 如果您想在没有 git 历史记录的情况下克隆仓库，可以使用 [degit](https://github.com/Rich-Harris/degit)。

打开浏览器并访问 [`http://localhost:3000/`](http://localhost:3000/)。

要安装 JavaScript 版本的入门项目，请在上述命令序列中使用 `javascript-starter.git`。

您也可以通过安装核心和支持包来从头开始一个新项目。请记住，您需要自行设置项目样板文件。至少，您需要这些依赖项：`@nestjs/core`、`@nestjs/common`、`rxjs` 和 `reflect-metadata`。查看这篇关于如何创建完整项目的简短文章：[5 steps to create a bare minimum NestJS app from scratch!](https://dev.to/micalevisk/5-steps-to-create-a-bare-minimum-nestjs-app-from-scratch-5c3b)。