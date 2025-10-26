### 文档

**Compodoc** 是一个为 Angular 应用打造的文档工具。由于 Nest 和 Angular 在项目和代码结构上非常相似，**Compodoc** 同样可以用于 Nest 应用程序。

#### 安装设置

在现有的 Nest 项目中设置 Compodoc 非常简单。首先，在你的操作系统终端中使用以下命令添加开发依赖：

```bash
$ npm i -D @compodoc/compodoc
```

#### 生成文档

使用以下命令生成项目文档（需要 npm 6 以支持 `npx`）。更多选项请参阅[官方文档](https://compodoc.app/guides/usage.html)。

```bash
$ npx @compodoc/compodoc -p tsconfig.json -s
```

打开你的浏览器并访问 [http://localhost:8080](http://localhost:8080)。你应该能看到一个初始的 Nest CLI 项目：

<figure><img src="/assets/documentation-compodoc-1.jpg" /></figure>
<figure><img src="/assets/documentation-compodoc-2.jpg" /></figure>

#### 参与贡献

你可以在此处参与并贡献 Compodoc 项目 [here](https://github.com/compodoc/compodoc)。