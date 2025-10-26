### SWC

[SWC](https://swc.rs/)（Speedy Web Compiler）是一个基于 Rust 的可扩展平台，可用于编译和打包。
在 Nest CLI 中使用 SWC 是一个极好且简单的方法，可以显著加快你的开发过程。

> info **提示** SWC 大约比默认的 TypeScript 编译器快 **20 倍**。

#### 安装

要开始使用，首先安装几个包：

```bash
$ npm i --save-dev @swc/cli @swc/core
```

#### 开始使用

安装过程完成后，你可以在 Nest CLI 中使用 `swc` 构建器，如下所示：

```bash
$ nest start -b swc
# 或者 nest start --builder swc
```

> info **提示** 如果你的仓库是 monorepo，请查看[此部分](/recipes/swc#monorepo)。

除了传递 `-b` 标志，你也可以在你的 `nest-cli.json` 文件中将 `compilerOptions.builder` 属性设置为 `"swc"`，像这样：

```json
{
  "compilerOptions": {
    "builder": "swc"
  }
}
```

要自定义构建器的行为，你可以传递一个包含两个属性 `type` (`"swc"`) 和 `options` 的对象，如下所示：

```json
"compilerOptions": {
  "builder": {
    "type": "swc",
    "options": {
      "swcrcPath": "infrastructure/.swcrc",
    }
  }
}
```

要在监听模式下运行应用程序，使用以下命令：

```bash
$ nest start -b swc -w
# 或者 nest start --builder swc --watch
```

#### 类型检查

SWC 本身不执行任何类型检查（与默认的 TypeScript 编译器相反），因此要开启类型检查，你需要使用 `--type-check` 标志：

```bash
$ nest start -b swc --type-check
```

此命令将指示 Nest CLI 在 `noEmit` 模式下与 SWC 一起运行 `tsc`，这将异步执行类型检查。同样，除了传递 `--type-check` 标志，你也可以在你的 `nest-cli.json` 文件中将 `compilerOptions.typeCheck` 属性设置为 `true`，像这样：

```json
{
  "compilerOptions": {
    "builder": "swc",
    "typeCheck": true
  }
}
```

#### CLI 插件 (SWC)

`--type-check` 标志将自动执行 **NestJS CLI 插件** 并生成一个序列化的元数据文件，然后该文件可以在运行时由应用程序加载。

#### SWC 配置

SWC 构建器已预先配置以匹配 NestJS 应用程序的要求。但是，你可以通过在根目录创建一个 `.swcrc` 文件并按需调整选项来自定义配置。

```json
{
  "$schema": "https://json.schemastore.org/swcrc",
  "sourceMaps": true,
  "jsc": {
    "parser": {
      "syntax": "typescript",
      "decorators": true,
      "dynamicImport": true
    },
    "baseUrl": "./"
  },
  "minify": false
}
```

#### Monorepo

如果你的仓库是 monorepo，那么你必须配置 `webpack` 使用 `swc-loader`，而不是使用 `swc` 构建器。

首先，安装所需的包：

```bash
$ npm i --save-dev swc-loader
```

安装完成后，在你的应用程序根目录创建一个 `webpack.config.js` 文件，内容如下：

```js
const swcDefaultConfig = require('@nestjs/cli/lib/compiler/defaults/swc-defaults').swcDefaultsFactory().swcOptions;

module.exports = {
  module: {
    rules: [
      {
        test: /\.ts$/,
        exclude: /node_modules/,
        use: {
          loader: 'swc-loader',
          options: swcDefaultConfig,
        },
      },
    ],
  },
};
```

#### Monorepo 和 CLI 插件

如果你使用 CLI 插件，`swc-loader` 不会自动加载它们。你必须创建一个单独的文件来手动加载它们。为此，在 `main.ts` 文件附近声明一个 `generate-metadata.ts` 文件，内容如下：

```ts
import { PluginMetadataGenerator } from '@nestjs/cli/lib/compiler/plugins/plugin-metadata-generator';
import { ReadonlyVisitor } from '@nestjs/swagger/dist/plugin';

const generator = new PluginMetadataGenerator();
generator.generate({
  visitors: [new ReadonlyVisitor({ introspectComments: true, pathToSource: __dirname })],
  outputDir: __dirname,
  watch: true,
  tsconfigPath: 'apps/<name>/tsconfig.app.json',
});
```

> info **提示** 在这个例子中，我们使用了 `@nestjs/swagger` 插件，但你可以使用任何你选择的插件。

`generate()` 方法接受以下选项：

|                    |                                                                                                |
| ------------------ | ---------------------------------------------------------------------------------------------- |
| `watch`            | 是否监听项目的更改。                                                                           |
| `tsconfigPath`     | `tsconfig.json` 文件的路径。相对于当前工作目录 (`process.cwd()`)。                             |
| `outputDir`        | 元数据文件保存目录的路径。                                                                     |
| `visitors`         | 用于生成元数据的访问器数组。                                                                   |
| `filename`         | 元数据文件的名称。默认为 `metadata.ts`。                                                       |
| `printDiagnostics` | 是否将诊断信息打印到控制台。默认为 `true`。                                                    |

最后，你可以在一个单独的终端窗口中运行以下命令来执行 `generate-metadata` 脚本：

```bash
$ npx ts-node src/generate-metadata.ts
# 或者 npx ts-node apps/{YOUR_APP}/src/generate-metadata.ts
```

#### 常见陷阱

如果你在应用程序中使用 TypeORM/MikroORM 或任何其他 ORM，你可能会遇到循环导入问题。SWC 不能很好地处理**循环导入**，所以你应该使用以下解决方法：

```typescript
@Entity()
export class User {
  @OneToOne(() => Profile, (profile) => profile.user)
  profile: Relation<Profile>; // <--- 这里使用 "Relation<>" 类型，而不是仅仅 "Profile"
}
```

> info **提示** `Relation` 类型从 `typeorm` 包中导出。

这样做可以防止属性的类型被保存在转译后代码的属性元数据中，从而避免循环依赖问题。

如果你的 ORM 没有提供类似的解决方法，你可以自己定义包装类型：

```typescript
/**
 * 用于规避 ESM 模块循环依赖问题的包装类型，
 * 该问题由反射元数据保存属性类型引起。
 */
export type WrapperType<T> = T; // WrapperType === Relation
```

对于你项目中所有的[循环依赖注入](/fundamentals/circular-dependency)，你还需要使用上述的自定义包装类型：

```typescript
@Injectable()
export class UserService {
  constructor(
    @Inject(forwardRef(() => ProfileService))
    private readonly profileService: WrapperType<ProfileService>,
  ) {};
}
```

### Jest + SWC

要在 Jest 中使用 SWC，你需要安装以下包：

```bash
$ npm i --save-dev jest @swc/core @swc/jest
```

安装完成后，更新 `package.json`/`jest.config.js` 文件（取决于你的配置），内容如下：

```json
{
  "jest": {
    "transform": {
      "^.+\\.(t|j)s?$": ["@swc/jest"]
    }
  }
}
```

此外，你需要在 `.swcrc` 文件中添加以下 `transform` 属性：`legacyDecorator`，`decoratorMetadata`：

```json
{
  "$schema": "https://json.schemastore.org/swcrc",
  "sourceMaps": true,
  "jsc": {
    "parser": {
      "syntax": "typescript",
      "decorators": true,
      "dynamicImport": true
    },
    "transform": {
      "legacyDecorator": true,
      "decoratorMetadata": true
    },
    "baseUrl": "./"
  },
  "minify": false
}
```

如果你在项目中使用 NestJS CLI 插件，你必须手动运行 `PluginMetadataGenerator`。请导航到[此部分](/recipes/swc#monorepo-and-cli-plugins)以了解更多信息。

### Vitest

[Vitest](https://vitest.dev/) 是一个快速、轻量级的测试运行器，旨在与 Vite 协同工作。它提供了一个现代化、快速且易于使用的测试解决方案，可以与 NestJS 项目集成。

#### 安装

要开始使用，首先安装所需的包：

```bash
$ npm i --save-dev vitest unplugin-swc @swc/core @vitest/coverage-v8
```

#### 配置

在你的应用程序根目录创建一个 `vitest.config.ts` 文件，内容如下：

```ts
import swc from 'unplugin-swc';
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    globals: true,
    root: './',
  },
  plugins: [
    // 这是用 SWC 构建测试文件所必需的
    swc.vite({
      // 显式设置模块类型，以避免从 `.swcrc` 配置文件中继承该值
      module: { type: 'es6' },
    }),
  ],
});
```

这个配置文件设置了 Vitest 环境、根目录和 SWC 插件。你还应该为 e2e 测试创建一个单独的配置文件，带有一个额外的 `include` 字段，用于指定测试路径的正则表达式：

```ts
import swc from 'unplugin-swc';
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    include: ['**/*.e2e-spec.ts'],
    globals: true,
    root: './',
  },
  plugins: [swc.vite()],
});
```

此外，你可以设置 `alias` 选项以在测试中支持 TypeScript 路径：

```ts
import swc from 'unplugin-swc';
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    include: ['**/*.e2e-spec.ts'],
    globals: true,
    alias: {
      '@src': './src',
      '@test': './test',
    },
    root: './',
  },
  resolve: {
    alias: {
      '@src': './src',
      '@test': './test',
    },
  },
  plugins: [swc.vite()],
});
```

#### 更新 E2E 测试中的导入

将所有使用 `import * as request from 'supertest'` 的 E2E 测试导入更改为 `import request from 'supertest'`。这是必要的，因为 Vitest 在与 Vite 捆绑时，期望 supertest 使用默认导入。在这种特定设置下，使用命名空间导入可能会导致问题。

最后，将你的 package.json 文件中的测试脚本更新如下：

```json
{
  "scripts": {
    "test": "vitest run",
    "test:watch": "vitest",
    "test:cov": "vitest run --coverage",
    "test:debug": "vitest --inspect-brk --inspect --logHeapUsage --threads=false",
    "test:e2e": "vitest run --config ./vitest.config.e2e.ts"
  }
}
```

这些脚本配置了 Vitest 用于运行测试、监听更改、生成代码覆盖率报告和调试。`test:e2e` 脚本专门用于使用自定义配置文件运行 E2E 测试。

通过此设置，你现在可以在 NestJS 项目中享受使用 Vitest 的好处，包括更快的测试执行速度和更现代化的测试体验。

> info **提示** 你可以在此[仓库](https://github.com/TrilonIO/nest-vitest)中查看一个可用的示例。