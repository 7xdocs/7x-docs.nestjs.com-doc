### 平台无关性

Nest 是一个与平台无关的框架。这意味着您可以开发**可重用的逻辑部件**，这些部件可以在不同类型的应用程序中使用。例如，大多数组件可以在不同的底层 HTTP 服务器框架（例如 Express 和 Fastify）之间无需更改即可重用，甚至可以在不同的*应用程序类型*之间重用（例如，HTTP 服务器框架、具有不同传输层的微服务以及 Web Sockets）。

#### 一次构建，随处使用

文档的**概述**部分主要展示了使用 HTTP 服务器框架的编码技术（例如，提供 REST API 的应用程序或提供 MVC 风格服务端渲染的应用程序）。然而，所有这些构建块都可以在不同的传输层之上使用（[微服务](/microservices/basics) 或 [websockets](/websockets/gateways)）。

此外，Nest 附带一个专用的 [GraphQL](/graphql/quick-start) 模块。您可以将 GraphQL 用作您的 API 层，与提供 REST API 相互替换。

另外，[应用上下文](/application-context) 功能有助于在 Nest 之上创建任何类型的 Node.js 应用程序——包括诸如 CRON 作业和 CLI 应用程序之类的东西。

Nest 立志成为一个功能齐全的 Node.js 应用程序平台，为您的应用程序带来更高级别的模块化和可重用性。一次构建，随处使用！