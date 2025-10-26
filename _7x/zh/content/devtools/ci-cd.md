### CI/CD 集成

> info **提示** 本章介绍 Nest 开发工具与 Nest 框架的集成。如果您正在寻找开发工具应用程序，请访问 [开发工具](https://devtools.nestjs.com) 网站。

CI/CD 集成仅对拥有 **[企业版](/settings)** 计划的用户开放。

您可以观看以下视频，了解 CI/CD 集成为何以及如何为您提供帮助：

<figure>
  <iframe
    width="1000"
    height="565"
    src="https://www.youtube.com/embed/r5RXcBrnEQ8"
    title="YouTube video player"
    frameBorder="0"
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
    allowFullScreen
  ></iframe>
</figure>

#### 发布图形

首先，我们来配置应用程序启动文件（`main.ts`），以使用 `GraphPublisher` 类（从 `@nestjs/devtools-integration` 导出 - 更多细节请参见上一章），配置如下：

```typescript
async function bootstrap() {
  const shouldPublishGraph = process.env.PUBLISH_GRAPH === "true";

  const app = await NestFactory.create(AppModule, {
    snapshot: true,
    preview: shouldPublishGraph,
  });

  if (shouldPublishGraph) {
    await app.init();

    const publishOptions = { ... } // NOTE: this options object will vary depending on the CI/CD provider you're using
    const graphPublisher = new GraphPublisher(app);
    await graphPublisher.publish(publishOptions);

    await app.close();
  } else {
    await app.listen(process.env.PORT ?? 3000);
  }
}
```

如我们所见，这里我们使用 `GraphPublisher` 将序列化的图形发布到中央注册表。`PUBLISH_GRAPH` 是一个自定义环境变量，用于控制是否应该发布图形（CI/CD 工作流），还是不发布（常规应用程序启动）。此外，我们在这里将 `preview` 属性设置为 `true`。启用此标志后，我们的应用程序将在预览模式下启动——这基本上意味着应用程序中所有控制器、增强器和提供者的构造函数（以及生命周期钩子）都不会执行。注意——这不是**必需的**，但会让我们的工作更简单，因为在这种情况下，我们在 CI/CD 流水线中运行应用程序时，实际上不必连接到数据库等。

`publishOptions` 对象会因您使用的 CI/CD 提供商而异。我们将在后面的部分中为您提供最流行的 CI/CD 提供商的说明。

图形成功发布后，您将在工作流视图中看到以下输出：

<figure><img src="/assets/devtools/graph-published-terminal.png" /></figure>

每次发布图形时，我们都应该在项目的对应页面中看到一个新条目：

<figure><img src="/assets/devtools/project.png" /></figure>

#### 报告

如果中央注册表中已存储相应的快照，开发工具会为每个构建生成报告。例如，如果您针对已发布图形的 `master` 分支创建一个 PR，那么应用程序将能够检测差异并生成报告。否则，将不会生成报告。

要查看报告，请导航到项目的对应页面（参见组织）。

<figure><img src="/assets/devtools/report.png" /></figure>

这对于识别代码审查过程中可能被忽视的变更特别有帮助。例如，假设有人更改了**深度嵌套的提供者**的作用域。这种变更可能不会立即被审查者发现，但通过开发工具，我们可以轻松发现此类变更，并确保它们是有意为之的。或者，如果我们从特定端点移除了一个守卫，它将在报告中显示为受影响的项。如果我们没有为该路由编写集成测试或端到端测试，我们可能不会注意到它不再受保护，而当我们发现时，可能已经太晚了。

同样，如果我们在**大型代码库**中工作，并将一个模块修改为全局模块，我们会看到图形中添加了多少条边，而在大多数情况下，这是我们做错了事情的信号。

#### 构建预览

对于每个已发布的图形，我们可以通过点击**预览**按钮回到过去，查看它之前的样子。此外，如果生成了报告，我们应该会在图形上看到突出显示的差异：

- 绿色节点表示新增的元素
- 浅白色节点表示更新的元素
- 红色节点表示删除的元素

参见下面的截图：

<figure><img src="/assets/devtools/nodes-selection.png" /></figure>

回溯功能使您能够通过比较当前图形和之前的图形来调查和排查问题。根据您的设置方式，每个拉取请求（甚至每个提交）在注册表中都会有相应的快照，因此您可以轻松回溯并查看发生了哪些变更。可以将开发工具视为具有理解 Nest 如何构建应用程序图形能力的 Git，并且能够**可视化**它。

#### 集成：GitHub Actions

首先，我们在项目的 `.github/workflows` 目录中创建一个新的 GitHub 工作流，例如将其命名为 `publish-graph.yml`。在该文件中，我们使用以下定义：

```yaml
name: Devtools

on:
  push:
    branches:
      - master
  pull_request:
    branches:
      - '*'

jobs:
  publish:
    if: github.actor!= 'dependabot[bot]'
    name: Publish graph
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '16'
          cache: 'npm'
      - name: Install dependencies
        run: npm ci
      - name: Setup Environment (PR)
        if: {{ '${{' }} github.event_name == 'pull_request' {{ '}}' }}
        shell: bash
        run: |
          echo "COMMIT_SHA={{ '${{' }} github.event.pull_request.head.sha {{ '}}' }}" >>\${GITHUB_ENV}
      - name: Setup Environment (Push)
        if: {{ '${{' }} github.event_name == 'push' {{ '}}' }}
        shell: bash
        run: |
          echo "COMMIT_SHA=\${GITHUB_SHA}" >> \${GITHUB_ENV}
      - name: Publish
        run: PUBLISH_GRAPH=true npm run start
        env:
          DEVTOOLS_API_KEY: CHANGE_THIS_TO_YOUR_API_KEY
          REPOSITORY_NAME: {{ '${{' }} github.event.repository.name {{ '}}' }}
          BRANCH_NAME: {{ '${{' }} github.head_ref || github.ref_name {{ '}}' }}
          TARGET_SHA: {{ '${{' }} github.event.pull_request.base.sha {{ '}}' }}
```

理想情况下，`DEVTOOLS_API_KEY` 环境变量应从 GitHub 密钥中获取，更多信息请参见 [此处](https://docs.github.com/en/actions/security-guides/encrypted-secrets#creating-encrypted-secrets-for-a-repository)。

此工作流将在每个针对 `master` 分支的拉取请求，或者直接提交到 `master` 分支时运行。您可以根据项目需求调整此配置。这里至关重要的是，我们为 `GraphPublisher` 类（运行时）提供了必要的环境变量。

但是，在开始使用此工作流之前，有一个变量需要更新——`DEVTOOLS_API_KEY`。我们可以在 [此页面](https://devtools.nestjs.com/settings/manage-api-keys) 上为我们的项目生成一个专用的 API 密钥。

最后，让我们再次导航到 `main.ts` 文件，并更新我们之前留空的 `publishOptions` 对象。

```typescript
const publishOptions = {
  apiKey: process.env.DEVTOOLS_API_KEY,
  repository: process.env.REPOSITORY_NAME,
  owner: process.env.GITHUB_REPOSITORY_OWNER,
  sha: process.env.COMMIT_SHA,
  target: process.env.TARGET_SHA,
  trigger: process.env.GITHUB_BASE_REF ? 'pull' : 'push',
  branch: process.env.BRANCH_NAME,
};
```

为了获得最佳的开发体验，请通过点击“集成 GitHub 应用”按钮为您的项目集成 **GitHub 应用**（参见下面的截图）。注意——这不是必需的。

<figure><img src="/assets/devtools/integrate-github-app.png" /></figure>

通过此集成，您将能够在拉取请求中直接查看预览/报告生成过程的状态：

<figure><img src="/assets/devtools/actions-preview.png" /></figure>

#### 集成：Gitlab 流水线

首先，我们在项目的根目录中创建一个新的 Gitlab CI 配置文件，例如将其命名为 `.gitlab-ci.yml`。在该文件中，我们使用以下定义：

```typescript
const publishOptions = {
  apiKey: process.env.DEVTOOLS_API_KEY,
  repository: process.env.REPOSITORY_NAME,
  owner: process.env.GITHUB_REPOSITORY_OWNER,
  sha: process.env.COMMIT_SHA,
  target: process.env.TARGET_SHA,
  trigger: process.env.GITHUB_BASE_REF ? 'pull' : 'push',
  branch: process.env.BRANCH_NAME,
};
```

> info **提示** 理想情况下，`DEVTOOLS_API_KEY` 环境变量应从密钥中获取。

此工作流将在每个针对 `master` 分支的拉取请求，或者直接提交到 `master` 分支时运行。您可以根据项目需求调整此配置。这里至关重要的是，我们为 `GraphPublisher` 类（运行时）提供了必要的环境变量。

但是，在开始使用此工作流之前，有一个变量（在此工作流定义中）需要更新——`DEVTOOLS_API_KEY`。我们可以在**此页面**上为我们的项目生成一个专用的 API 密钥。

最后，让我们再次导航到 `main.ts` 文件，并更新我们之前留空的 `publishOptions` 对象。

```yaml
image: node:16

stages:
  - build

cache:
  key:
    files:
      - package-lock.json
  paths:
    - node_modules/

workflow:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      when: always
    - if: $CI_COMMIT_BRANCH == "master" && $CI_PIPELINE_SOURCE == "push"
      when: always
    - when: never

install_dependencies:
  stage: build
  script:
    - npm ci

publish_graph:
  stage: build
  needs:
    - install_dependencies
  script: npm run start
  variables:
    PUBLISH_GRAPH: 'true'
    DEVTOOLS_API_KEY: 'CHANGE_THIS_TO_YOUR_API_KEY'
```

#### 其他 CI/CD 工具

Nest 开发工具的 CI/CD 集成可与您选择的任何 CI/CD 工具一起使用（例如 [Bitbucket Pipelines](https://bitbucket.org/product/features/pipelines)、[CircleCI](https://circleci.com/) 等），因此不要局限于我们这里描述的提供商。

查看以下 `publishOptions` 对象配置，了解为给定的提交/构建/PR 发布图形所需的信息。

```typescript
const publishOptions = {
  apiKey: process.env.DEVTOOLS_API_KEY,
  repository: process.env.CI_PROJECT_NAME,
  owner: process.env.CI_PROJECT_ROOT_NAMESPACE,
  sha: process.env.CI_COMMIT_SHA,
  target: process.env.CI_MERGE_REQUEST_DIFF_BASE_SHA,
  trigger: process.env.CI_MERGE_REQUEST_DIFF_BASE_SHA ? 'pull' : 'push',
  branch: process.env.CI_COMMIT_BRANCH ?? process.env.CI_MERGE_REQUEST_SOURCE_BRANCH_NAME,
};
```

这些信息中的大部分是通过 CI/CD 内置环境变量提供的（参见 [CircleCI 内置环境变量列表](https://circleci.com/docs/variables/#built-in-environment-variables) 和 [Bitbucket 变量](https://support.atlassian.com/bitbucket-cloud/docs/variables-and-secrets/)）。

关于发布图形的流水线配置，我们建议使用以下触发器：

- `push` 事件——仅当当前分支代表部署环境时，例如 `master`、`main`、`staging`、`production` 等。
- `pull request` 事件——始终触发，或者当**目标分支**代表部署环境时（见上文）