### SQL (Sequelize)

##### 本章仅适用于 TypeScript

> **警告** 在本文中，您将学习如何使用自定义组件基于 **Sequelize** 包从零开始创建一个 `DatabaseModule`。因此，这种技术包含了许多可以通过使用开箱即用的专用 `@nestjs/sequelize` 包来避免的开销。了解更多信息，请参阅[这里](/techniques/database#sequelize-integration)。

[Sequelize](https://github.com/sequelize/sequelize) 是一个用原生 JavaScript 编写的流行对象关系映射器（ORM），但有一个 [sequelize-typescript](https://github.com/RobinBuschmann/sequelize-typescript) TypeScript 包装器，它为基础的 sequelize 提供了一系列装饰器和其他额外功能。

#### 入门

要开始使用这个库，我们必须安装以下依赖项：

```bash
$ npm install --save sequelize sequelize-typescript mysql2
$ npm install --save-dev @types/sequelize
```

我们需要做的第一步是创建一个 **Sequelize** 实例，并将一个配置对象传递给构造函数。此外，我们需要添加所有模型（另一种方法是使用 `modelPaths` 属性）并 `sync()` 我们的数据库表。

```typescript
@@filename(database.providers)
import { Sequelize } from 'sequelize-typescript';
import { Cat } from '../cats/cat.entity';

export const databaseProviders = [
  {
    provide: 'SEQUELIZE',
    useFactory: async () => {
      const sequelize = new Sequelize({
        dialect: 'mysql',
        host: 'localhost',
        port: 3306,
        username: 'root',
        password: 'password',
        database: 'nest',
      });
      sequelize.addModels([Cat]);
      await sequelize.sync();
      return sequelize;
    },
  },
];
```

> info **提示** 遵循最佳实践，我们在一个单独的文件中声明了自定义提供者，该文件具有 `*.providers.ts` 后缀。

然后，我们需要导出这些提供者，以使应用程序的其余部分可以**访问**它们。

```typescript
import { Module } from '@nestjs/common';
import { databaseProviders } from './database.providers';

@Module({
  providers: [...databaseProviders],
  exports: [...databaseProviders],
})
export class DatabaseModule {}
```

现在我们可以使用 `@Inject()` 装饰器注入 `Sequelize` 对象。任何依赖于 `Sequelize` 异步提供者的类都将等待 `Promise` 被解析。

#### 模型注入

在 [Sequelize](https://github.com/sequelize/sequelize) 中，**模型**定义了数据库中的一个表。该类的实例代表数据库中的一行。首先，我们至少需要一个实体：

```typescript
@@filename(cat.entity)
import { Table, Column, Model } from 'sequelize-typescript';

@Table
export class Cat extends Model {
  @Column
  name: string;

  @Column
  age: number;

  @Column
  breed: string;
}
```

`Cat` 实体属于 `cats` 目录。该目录代表 `CatsModule`。现在是时候创建一个**存储库**提供者了：

```typescript
@@filename(cats.providers)
import { Cat } from './cat.entity';

export const catsProviders = [
  {
    provide: 'CATS_REPOSITORY',
    useValue: Cat,
  },
];
```

> warning **警告** 在实际应用程序中，应避免使用**魔法字符串**。`CATS_REPOSITORY` 和 `SEQUELIZE` 都应保存在单独的 `constants.ts` 文件中。

在 Sequelize 中，我们使用静态方法来操作数据，因此我们在这里创建了一个**别名**。

现在我们可以使用 `@Inject()` 装饰器将 `CATS_REPOSITORY` 注入到 `CatsService` 中：

```typescript
@@filename(cats.service)
import { Injectable, Inject } from '@nestjs/common';
import { CreateCatDto } from './dto/create-cat.dto';
import { Cat } from './cat.entity';

@Injectable()
export class CatsService {
  constructor(
    @Inject('CATS_REPOSITORY')
    private catsRepository: typeof Cat
  ) {}

  async findAll(): Promise<Cat[]> {
    return this.catsRepository.findAll<Cat>();
  }
}
```

数据库连接是**异步的**，但 Nest 使这个过程对最终用户完全不可见。`CATS_REPOSITORY` 提供者正在等待数据库连接，而 `CatsService` 会延迟到存储库准备就绪。整个应用程序可以在每个类实例化后启动。

这是一个最终的 `CatsModule`：

```typescript
@@filename(cats.module)
import { Module } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';
import { catsProviders } from './cats.providers';
import { DatabaseModule } from '../database/database.module';

@Module({
  imports: [DatabaseModule],
  controllers: [CatsController],
  providers: [
    CatsService,
    ...catsProviders,
  ],
})
export class CatsModule {}
```

> info **提示** 不要忘记将 `CatsModule` 导入到根 `AppModule` 中。