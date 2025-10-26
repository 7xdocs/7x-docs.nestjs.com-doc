### 部署

当你准备将 NestJS 应用程序部署到生产环境时，可以采取一些关键步骤来确保其尽可能高效地运行。在本指南中，我们将探讨必要的技巧和最佳实践，以帮助你成功部署 NestJS 应用程序。

#### 前提条件

在部署 NestJS 应用程序之前，请确保你具备：

- 一个可供部署的、功能正常的 NestJS 应用程序。
- 对可以托管应用程序的部署平台或服务器的访问权限。
- 为应用程序设置了所有必要的环境变量。
- 任何所需的服务（例如数据库）已设置并准备就绪。
- 在部署平台上至少安装了 Node.js 的 LTS 版本。

> info **提示** 如果你正在寻找基于云的平台来部署你的 NestJS 应用程序，请查看 [Mau](https://mau.nestjs.com/ 'Deploy Nest')，这是我们在 AWS 上部署 NestJS 应用程序的官方平台。使用 Mau，部署你的 NestJS 应用程序就像点击几下按钮并运行一条命令一样简单：
>
> ```bash
> $ npm install -g @nestjs/mau
> $ mau deploy
> ```
>
> 部署完成后，你的 NestJS 应用程序将在几秒钟内在 AWS 上启动并运行！

#### 构建你的应用程序

要构建你的 NestJS 应用程序，你需要将 TypeScript 代码编译成 JavaScript。此过程会生成一个包含已编译文件的 `dist` 目录。你可以通过运行以下命令来构建你的应用程序：

```bash
$ npm run build
```

此命令通常在底层运行 `nest build` 命令，这基本上是 TypeScript 编译器的一个包装器，附带一些附加功能（资源复制等）。如果你有自定义的构建脚本，也可以直接运行它。另外，对于 NestJS CLI 单体仓库，请确保将要构建的项目名称作为参数传递（`npm run build my-app`）。

成功编译后，你应该在项目根目录中看到一个包含已编译文件的 `dist` 目录，入口点是 `main.js`。如果你在项目的根目录中有任何 `.ts` 文件（并且你的 `tsconfig.json` 配置为编译它们），它们也将被复制到 `dist` 目录中，这会稍微修改目录结构（你将得到 `dist/src/main.js` 而不是 `dist/main.js`，在配置服务器时请记住这一点）。

#### 生产环境

你的生产环境是外部用户可以访问你的应用程序的地方。这可以是基于云的平台，如 [AWS](https://aws.amazon.com/)（使用 EC2、ECS 等）、[Azure](https://azure.microsoft.com/) 或 [Google Cloud](https://cloud.google.com/)，甚至可以是你自己管理的专用服务器，例如 [Hetzner](https://www.hetzner.com/)。

为了简化部署过程并避免手动设置，你可以使用像 [Mau](https://mau.nestjs.com/ 'Deploy Nest') 这样的服务，这是我们在 AWS 上部署 NestJS 应用程序的官方平台。有关更多详细信息，请查看[此部分](todo)。

使用**基于云的平台**或像 [Mau](https://mau.nestjs.com/ 'Deploy Nest') 这样的服务的一些优点包括：

- 可扩展性：随着用户群的增长，轻松扩展你的应用程序。
- 安全性：受益于内置的安全功能和合规性认证。
- 监控：实时监控应用程序的性能和健康状况。
- 可靠性：通过高正常运行时间保证，确保你的应用程序始终可用。

另一方面，基于云的平台通常比自我托管更昂贵，并且你可能对底层基础设施的控制较少。如果你正在寻找更具成本效益的解决方案并且拥有自行管理服务器的技术专长，简单的 VPS 可能是一个不错的选择，但请记住，你需要手动处理服务器维护、安全和备份等任务。

#### NODE_ENV=production

虽然在 Node.js 和 NestJS 中，开发和生产环境在技术上没有区别，但在生产环境中运行应用程序时，将 `NODE_ENV` 环境变量设置为 `production` 是一个很好的做法，因为生态系统中的一些库可能会根据此变量表现出不同的行为（例如，启用或禁用调试输出等）。

你可以像这样在启动应用程序时设置 `NODE_ENV` 环境变量：

```bash
$ NODE_ENV=production node dist/main.js
```

或者只需在你的云提供商/Mau 仪表板中设置它。

#### 运行你的应用程序

要在生产环境中运行你的 NestJS 应用程序，只需使用以下命令：

```bash
$ node dist/main.js # 根据你的入口点位置调整此命令
```

此命令启动你的应用程序，它将监听指定的端口（通常默认为 `3000`）。确保这与你已在应用程序中配置的端口匹配。

或者，你可以使用 `nest start` 命令。此命令是 `node dist/main.js` 的一个包装器，但它有一个关键区别：它在启动应用程序之前自动运行 `nest build`，因此你无需手动执行 `npm run build`。

#### 健康检查

健康检查对于在生产环境中监控 NestJS 应用程序的健康状况和状态至关重要。通过设置健康检查端点，你可以定期验证你的应用程序是否按预期运行，并在问题变得严重之前做出响应。

在 NestJS 中，你可以使用 **@nestjs/terminus** 包轻松实现健康检查，该包提供了一个强大的工具来添加健康检查，包括数据库连接、外部服务和自定义检查。

查看[本指南](/recipes/terminus)了解如何在你的 NestJS 应用程序中实现健康检查，并确保你的应用程序始终受到监控且响应迅速。

#### 日志记录

日志记录对于任何生产就绪的应用程序都至关重要。它有助于跟踪错误、监控行为和排查问题。在 NestJS 中，你可以使用内置的记录器轻松管理日志记录，或者如果需要更高级的功能，可以选择外部库。

日志记录的最佳实践：

- 记录错误，而非异常：专注于记录详细的错误消息，以加快调试和问题解决速度。
- 避免敏感数据：切勿记录敏感信息（如密码或令牌）以保护安全。
- 使用关联 ID：在分布式系统中，在日志中包含唯一标识符（如关联 ID）以跟踪不同服务的请求。
- 使用日志级别：按严重性（例如 `info`、`warn`、`error`）对日志进行分类，并在生产环境中禁用调试或详细日志以减少噪音。

> info **提示** 如果你正在使用 [AWS](https://aws.amazon.com/)（通过 [Mau](https://mau.nestjs.com/ 'Deploy Nest') 或直接使用），请考虑使用 JSON 日志记录，以便更轻松地解析和分析你的日志。

对于分布式应用程序，使用集中式日志记录服务（如 ElasticSearch、Loggly 或 Datadog）非常有用。这些工具提供强大的功能，如日志聚合、搜索和可视化，使你更容易监控和分析应用程序的性能和行为。

#### 纵向或横向扩展

有效扩展你的 NestJS 应用程序对于处理增加的流量和确保最佳性能至关重要。主要有两种扩展策略：**纵向扩展**和**横向扩展**。了解这些方法将帮助你设计应用程序以有效管理负载。

**纵向扩展**，通常称为"向上扩展"，涉及增加单个服务器的资源以提升其性能。这可能意味着为你现有的机器增加更多的 CPU、RAM 或存储。以下是一些需要考虑的要点：

- 简单性：纵向扩展通常更简单，因为你只需要升级现有服务器，而无需管理多个实例。
- 局限性：单个机器的扩展能力存在物理限制。一旦达到最大容量，你可能需要考虑其他选项。
- 成本效益：对于流量适中的应用程序，纵向扩展可能更具成本效益，因为它减少了对额外基础设施的需求。

示例：如果你的 NestJS 应用程序托管在虚拟机上，并且你发现它在高峰时段运行缓慢，你可以将你的 VM 升级到具有更多资源的更大实例。要升级你的 VM，只需导航到你当前提供商的仪表板并选择更大的实例类型。

**横向扩展**，或"向外扩展"，涉及添加更多服务器或实例以分布负载。此策略在云环境中广泛使用，对于预期高流量的应用程序至关重要。以下是好处和考虑因素：

- 增加容量：通过添加更多应用程序实例，你可以处理更多的并发用户而不会降低性能。
- 冗余：横向扩展提供冗余，因为一个服务器的故障不会导致整个应用程序宕机。流量可以在剩余的服务器之间重新分配。
- 负载均衡：为了有效管理多个实例，请使用负载均衡器（如 Nginx 或 AWS Elastic Load Balancing）将传入流量均匀分布到你的服务器上。

示例：对于遇到高流量的 NestJS 应用程序，你可以在云环境中部署多个应用程序实例，并使用负载均衡器路由请求，确保没有任何一个实例成为瓶颈。

使用容器化技术（如 [Docker](https://www.docker.com/)）和容器编排平台（如 [Kubernetes](https://kubernetes.io/)），这个过程非常简单。此外，你可以利用云特定的负载均衡器，如 [AWS Elastic Load Balancing](https://aws.amazon.com/elasticloadbalancing/) 或 [Azure Load Balancer](https://azure.microsoft.com/en-us/services/load-balancer/)，将流量分布到你的应用程序实例。

> info **提示** [Mau](https://mau.nestjs.com/ 'Deploy Nest') 在 AWS 上提供对横向扩展的内置支持，使你只需点击几下即可轻松部署多个 NestJS 应用程序实例并管理它们。

#### 其他一些技巧

在部署 NestJS 应用程序时，还有几个技巧需要记住：

- **安全性**：确保你的应用程序安全并受到保护，免受 SQL 注入、XSS 等常见威胁。有关更多详细信息，请参阅"安全"类别。
- **监控**：使用监控工具，如 [Prometheus](https://prometheus.io/) 或 [New Relic](https://newrelic.com/)，来跟踪应用程序的性能和健康状况。如果你使用的是云提供商/Mau，他们可能提供内置的监控服务（如 [AWS CloudWatch](https://aws.amazon.com/cloudwatch/) 等）。
- **不要硬编码环境变量**：避免在代码中硬编码敏感信息，如 API 密钥、密码或令牌。使用环境变量或密钥管理器来安全地存储和访问这些值。
- **备份**：定期备份数据，以防止在发生事故时数据丢失。
- **自动化部署**：使用 CI/CD 流水线自动化部署过程，并确保跨环境的一致性。
- **速率限制**：实施速率限制以防止滥用并保护你的应用程序免受 DDoS 攻击。查看[速率限制章节](/security/rate-limiting)了解更多详细信息，或使用像 [AWS WAF](https://aws.amazon.com/waf/) 这样的服务来获得高级保护。

#### 将你的应用程序 Docker 化

[Docker](https://www.docker.com/) 是一个使用容器化的平台，允许开发人员将应用程序及其依赖项打包到一个称为容器的标准化单元中。容器轻量、可移植且隔离，非常适合在各种环境中（从本地开发到生产环境）部署应用程序。

将你的 NestJS 应用程序 Docker 化的好处：

- 一致性：Docker 确保你的应用程序在任何机器上以相同的方式运行，消除了"在我机器上可以运行"的问题。
- 隔离性：每个容器在其隔离的环境中运行，防止依赖项之间的冲突。
- 可扩展性：通过在不同机器或云实例上运行多个容器，Docker 使扩展你的应用程序变得容易。
- 可移植性：容器可以轻松在环境之间移动，使得在不同平台上部署应用程序变得简单。

要安装 Docker，请遵循[官方网站](https://www.docker.com/get-started)上的说明。安装 Docker 后，你可以在 NestJS 项目中创建一个 `Dockerfile`，以定义构建容器镜像的步骤。

`Dockerfile` 是一个文本文件，包含 Docker 用于构建容器镜像的指令。

以下是 NestJS 应用程序的示例 Dockerfile：

```bash
# Use the official Node.js image as the base image
FROM node:20

# Set the working directory inside the container
WORKDIR /usr/src/app

# Copy package.json and package-lock.json to the working directory
COPY package*.json ./

# Install the application dependencies
RUN npm install

# Copy the rest of the application files
COPY . .

# Build the NestJS application
RUN npm run build

# Expose the application port
EXPOSE 3000

# Command to run the application
CMD ["node", "dist/main"]
```

> info **提示** 确保将 `node:20` 替换为你项目中使用的适当 Node.js 版本。你可以在[官方 Docker Hub 仓库](https://hub.docker.com/_/node)上找到可用的 Node.js Docker 镜像。

这是一个基本的 Dockerfile，它设置了 Node.js 环境，安装应用程序依赖项，构建 NestJS 应用程序并运行它。你可以根据项目要求自定义此文件（例如，使用不同的基础镜像，优化构建过程，仅安装生产依赖项等）。

让我们也创建一个 `.dockerignore` 文件，以指定 Docker 在构建镜像时应忽略哪些文件和目录。在你的项目根目录中创建一个 `.dockerignore` 文件：

```bash
node_modules
dist
*.log
*.md
.git
```

此文件确保不必要的文件不包含在容器镜像中，使其保持轻量。现在你已经设置好了 Dockerfile，你可以构建你的 Docker 镜像。打开终端，导航到你的项目目录，并运行以下命令：

```bash
docker build -t my-nestjs-app .
```

在此命令中：

- `-t my-nestjs-app`：用名称 `my-nestjs-app` 标记镜像。
- `.`：表示当前目录作为构建上下文。

构建镜像后，你可以将其作为容器运行。执行以下命令：

```bash
docker run -p 3000:3000 my-nestjs-app
```

在此命令中：

- `-p 3000:3000`：将主机上的 3000 端口映射到容器中的 3000 端口。
- `my-nestjs-app`：指定要运行的镜像。

你的 NestJS 应用程序现在应该在一个 Docker 容器内运行。

如果你想将 Docker 镜像部署到云提供商或与他人共享，你需要将其推送到 Docker 注册表（如 [Docker Hub](https://hub.docker.com/)、[AWS ECR](https://aws.amazon.com/ecr/) 或 [Google Container Registry](https://cloud.google.com/container-registry)）。

一旦你决定了注册表，你可以按照以下步骤推送你的镜像：

```bash
docker login # 登录到你的 Docker 注册表
docker tag my-nestjs-app your-dockerhub-username/my-nestjs-app # 标记你的镜像
docker push your-dockerhub-username/my-nestjs-app # 推送你的镜像
```

将 `your-dockerhub-username` 替换为你的 Docker Hub 用户名或适当的注册表 URL。推送镜像后，你可以在任何机器上拉取它并将其作为容器运行。

像 AWS、Azure 和 Google Cloud 这样的云提供商提供托管的容器服务，可以简化大规模容器的部署和管理。这些服务提供自动扩展、负载均衡和监控等功能，使你在生产环境中运行 NestJS 应用程序变得更加容易。

#### 使用 Mau 轻松部署

[Mau](https://mau.nestjs.com/ 'Deploy Nest') 是我们在 [AWS](https://aws.amazon.com/) 上部署 NestJS 应用程序的官方平台。如果你不准备手动管理你的基础设施（或者只是想节省时间），Mau 是你的完美解决方案。

使用 Mau，配置和维护你的基础设施就像点击几下按钮一样简单。Mau 设计得简单直观，因此你可以专注于构建应用程序，而无需担心底层基础设施。在底层，我们使用 **Amazon Web Services** 为你提供一个强大可靠的平台，同时抽象掉 AWS 的所有复杂性。我们为你处理所有繁重的工作，因此你可以专注于构建应用程序和发展业务。

[Mau](https://mau.nestjs.com/ 'Deploy Nest') 非常适合初创公司、中小型企业、大型企业以及希望快速启动运行而不想花费大量时间学习和管理基础设施的开发人员。它非常易于使用，你可以在几分钟内启动并运行你的基础设施。它还在幕后利用 AWS，让你享有 AWS 的所有优势，而无需管理其复杂性。

<figure><img src="/assets/mau-metrics.png" /></figure>

使用 [Mau](https://mau.nestjs.com/ 'Deploy Nest')，你可以：

- 只需点击几下即可部署你的 NestJS 应用程序（API、微服务等）。
- 配置**数据库**，例如：
  - PostgreSQL
  - MySQL
  - MongoDB (DocumentDB)
  - Redis
  - 更多
- 设置代理服务，如：
  - RabbitMQ
  - Kafka
  - NATS
- 部署计划任务（**CRON 作业**）和后台工作程序。
- 部署 lambda 函数和无服务器应用程序。
- 设置 **CI/CD 流水线**以实现自动化部署。
- 以及更多！

要使用 Mau 部署你的 NestJS 应用程序，只需运行以下命令：

```bash
$ npm install -g @nestjs/mau
$ mau deploy
```

立即注册并[使用 Mau 部署](https://mau.nestjs.com/ 'Deploy Nest')，让你的 NestJS 应用程序在几分钟内在 AWS 上启动并运行！