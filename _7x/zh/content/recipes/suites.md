### Suites (前身为 Automock)

Suites 是一个具有观点且灵活的测试元框架，旨在提升后端系统的软件测试体验。通过将多种测试工具整合到一个统一的框架中，Suites 简化了可靠测试的创建过程，有助于确保开发出高质量的软件。

> info **提示** `Suites` 是一个第三方包，并非由 NestJS 核心团队维护。请将有关该库的任何问题报告到[相应的代码仓库](https://github.com/suites-dev/suites)。

#### 介绍

控制反转（IoC）是 NestJS 框架中的一个基本原则，它支持模块化、可测试的架构。虽然 NestJS 提供了用于创建测试模块的内置工具，但 Suites 提供了一种替代方法，强调测试隔离的单元或小组单元。Suites 使用一个用于依赖项的虚拟容器，其中自动生成模拟对象，无需在 IoC（或 DI）容器中手动将每个提供者替换为模拟对象。这种方法可以替代或与 NestJS 的 `Test.createTestingModule` 方法一起使用，根据您的需求为单元测试提供更大的灵活性。

#### 安装

要在 NestJS 中使用 Suites，请安装必要的包：

```bash
$ npm i -D @suites/unit @suites/di.nestjs @suites/doubles.jest
```

> info **提示** `Suites` 也支持 Vitest 和 Sinon 作为测试替身，分别是 `@suites/doubles.vitest` 和 `@suites/doubles.sinon`。

#### 示例和模块设置

考虑一个为 `CatsService` 设置的模块，该模块包含 `CatsApiService`、`CatsDAL`、`HttpClient` 和 `Logger`。这将是我们本指南中示例的基础：

```typescript
@@filename(cats.module)
import { HttpModule } from '@nestjs/axios';
import { PrismaModule } from '../prisma.module';

@Module({
  imports: [HttpModule.register({ baseUrl: 'https://api.cats.com/' }), PrismaModule],
  providers: [CatsService, CatsApiService, CatsDAL, Logger],
  exports: [CatsService],
})
export class CatsModule {}
```

`HttpModule` 和 `PrismaModule` 都在向宿主模块导出提供者。

让我们开始隔离测试 `CatsHttpService`。该服务负责从 API 获取猫的数据并记录操作。

```typescript
@@filename(cats-http.service)
@Injectable()
export class CatsHttpService {
  constructor(private httpClient: HttpClient, private logger: Logger) {}

  async fetchCats(): Promise<Cat[]> {
    this.logger.log('Fetching cats from the API');
    const response = await this.httpClient.get('/cats');
    return response.data;
  }
}
```

我们希望隔离 `CatsHttpService` 并模拟其依赖项 `HttpClient` 和 `Logger`。Suites 允许我们使用 `TestBed` 中的 `.solitary()` 方法轻松实现这一点。

```typescript
@@filename(cats-http.service.spec)
import { TestBed, Mocked } from '@suites/unit';

describe('Cats Http Service Unit Test', () => {
  let catsHttpService: CatsHttpService;
  let httpClient: Mocked<HttpClient>;
  let logger: Mocked<Logger>;

  beforeAll(async () => {
    // Isolate CatsHttpService and mock HttpClient and Logger
    const { unit, unitRef } = await TestBed.solitary(CatsHttpService).compile();

    catsHttpService = unit;
    httpClient = unitRef.get(HttpClient);
    logger = unitRef.get(Logger);
  });

  it('should fetch cats from the API and log the operation', async () => {
    const catsFixtures: Cat[] = [{ id: 1, name: 'Catty' }, { id: 2, name: 'Mitzy' }];
    httpClient.get.mockResolvedValue({ data: catsFixtures });

    const cats = await catsHttpService.fetchCats();

    expect(logger.log).toHaveBeenCalledWith('Fetching cats from the API');
    expect(httpClient.get).toHaveBeenCalledWith('/cats');
    expect(cats).toEqual<Cat[]>(catsFixtures);
  });
});
```

在上面的示例中，Suites 使用 `TestBed.solitary()` 自动模拟了 `CatsHttpService` 的依赖项。这使得设置更容易，因为您不必手动模拟每个依赖项。

- 依赖项的自动模拟：Suites 为被测试单元的所有依赖项生成模拟对象。
- 模拟对象的初始行为：最初，这些模拟对象没有任何预定义的行为。您需要根据测试需求指定它们的行为。
- `unit` 和 `unitRef` 属性：
  - `unit` 指的是被测试类的实际实例，包含其模拟的依赖项。
  - `unitRef` 是一个引用，允许您访问模拟的依赖项。

#### 使用 `TestingModule` 测试 `CatsApiService`

对于 `CatsApiService`，我们希望确保 `HttpModule` 在 `CatsModule` 宿主模块中正确导入和配置。这包括验证 `Axios` 的基本 URL（和其他配置）是否设置正确。

在这种情况下，我们不会使用 Suites；相反，我们将使用 Nest 的 `TestingModule` 来测试 `HttpModule` 的实际配置。我们将利用 `nock` 来模拟 HTTP 请求，而不在此场景中模拟 `HttpClient`。

```typescript
@@filename(cats-api.service)
import { HttpClient } from '@nestjs/axios';

@Injectable()
export class CatsApiService {
  constructor(private httpClient: HttpClient) {}

  async getCatById(id: number): Promise<Cat> {
    const response = await this.httpClient.get(`/cats/${id}`);
    return response.data;
  }
}
```

我们需要使用真实的、未模拟的 `HttpClient` 来测试 `CatsApiService`，以确保 `Axios` (http) 的 DI 和配置正确。这涉及到导入 `CatsModule` 并使用 `nock` 进行 HTTP 请求模拟。

```typescript
@@filename(cats-api.service.integration.test)
import { Test } from '@nestjs/testing';
import * as nock from 'nock';

describe('Cats Api Service Integration Test', () => {
  let catsApiService: CatsApiService;

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({
      imports: [CatsModule],
    }).compile();

    catsApiService = moduleRef.get(CatsApiService);
  });

  afterEach(() => {
    nock.cleanAll();
  });

  it('should fetch cat by id using real HttpClient', async () => {
    const catFixture: Cat = { id: 1, name: 'Catty' };

    nock('https://api.cats.com') // Making this URL identical to the one in HttpModule registration
      .get('/cats/1')
      .reply(200, catFixture);

    const cat = await catsApiService.getCatById(1);
    expect(cat).toEqual<Cat>(catFixture);
  });
});
```

#### 社交测试示例

接下来，让我们测试 `CatsService`，它依赖于 `CatsApiService` 和 `CatsDAL`。我们将模拟 `CatsApiService` 并暴露 `CatsDAL`。

```typescript
@@filename(cats.dal)
import { PrismaClient } from '@prisma/client';

@Injectable()
export class CatsDAL {
  constructor(private prisma: PrismaClient) {}

  async saveCat(cat: Cat): Promise<Cat> {
    return this.prisma.cat.create({data: cat});
  }
}
```

接下来是 `CatsService`，它依赖于 `CatsApiService` 和 `CatsDAL`：

```typescript
@@filename(cats.service)
@Injectable()
export class CatsService {
  constructor(
    private catsApiService: CatsApiService,
    private catsDAL: CatsDAL
  ) {}

  async getAndSaveCat(id: number): Promise<Cat> {
    const cat = await this.catsApiService.getCatById(id);
    return this.catsDAL.saveCat(cat);
  }
}
```

现在，让我们使用 Suites 进行社交测试来测试 `CatsService`：

```typescript
@@filename(cats.service.spec)
import { TestBed, Mocked } from '@suites/unit';
import { PrismaClient } from '@prisma/client';

describe('Cats Service Sociable Unit Test', () => {
  let catsService: CatsService;
  let prisma: Mocked<PrismaClient>;
  let catsApiService: Mocked<CatsApiService>;

  beforeAll(async () => {
    // Sociable test setup, exposing CatsDAL and mocking CatsApiService
    const { unit, unitRef } = await TestBed.sociable(CatsService)
      .expose(CatsDAL)
      .mock(CatsApiService)
      .final({ getCatById: async () => ({ id: 1, name: 'Catty' })})
      .compile();

    catsService = unit;
    prisma = unitRef.get(PrismaClient);
  });

  it('should get cat by id and save it', async () => {
    const catFixture: Cat = { id: 1, name: 'Catty' };
    prisma.cat.create.mockResolvedValue(catFixture);

    const savedCat = await catsService.getAndSaveCat(1);

    expect(prisma.cat.create).toHaveBeenCalledWith({ data: catFixture });
    expect(savedCat).toEqual(catFixture);
  });
});
```

在此示例中，我们使用 `.sociable()` 方法来设置测试环境。我们使用 `.expose()` 方法允许与 `CatsDAL` 进行真实交互，同时使用 `.mock()` 方法模拟 `CatsApiService`。`.final()` 方法为 `CatsApiService` 建立了固定的行为，确保测试结果的一致性。

这种方法强调使用与 `CatsDAL` 的真实交互来测试 `CatsService`，这涉及到处理 `Prisma`。Suites 将按原样使用 `CatsDAL`，只有其依赖项（如 `Prisma`）在这种情况下会被模拟。

需要注意的是，这种方法**仅用于验证行为**，并且与加载整个测试模块不同。社交测试对于确认单元与其直接依赖项隔离时的行为非常有价值，特别是当您希望关注单元的行为和交互时。

#### 集成测试和数据库

对于 `CatsDAL`，可以针对真实数据库（如 SQLite 或 PostgreSQL）进行测试（例如，使用 Docker Compose）。但是，在此示例中，我们将模拟 `Prisma` 并专注于社交测试。模拟 `Prisma` 的原因是为了避免 I/O 操作，并专注于隔离状态下 `CatsService` 的行为。也就是说，您也可以进行具有真实 I/O 操作和实时数据库的测试。

#### 社交单元测试、集成测试和模拟

- 社交单元测试：这些测试侧重于测试单元之间的交互和行为，同时模拟它们更深层次的依赖项。在此示例中，我们模拟了 `Prisma` 并暴露了 `CatsDAL`。

- 集成测试：这些测试涉及真实的 I/O 操作和完全配置的依赖注入（DI）设置。使用 `HttpModule` 和 `nock` 测试 `CatsApiService` 被视为集成测试，因为它验证了 `HttpClient` 的实际配置和交互。在这种情况下，我们将使用 Nest 的 `TestingModule` 来加载实际的模块配置。

**使用模拟时要谨慎。** 务必测试 I/O 操作和 DI 配置（尤其是在涉及 HTTP 或数据库交互时）。在通过集成测试验证了这些组件之后，您可以放心地在社交单元测试中模拟它们，以专注于行为和交互。Suites 社交测试旨在验证单元与其直接依赖项隔离时的行为，而集成测试则确保整个系统配置和 I/O 操作正常运行。

#### 测试 IoC 容器注册

验证您的 DI 容器是否正确配置至关重要，可以防止运行时错误。这包括确保所有提供者、服务和模块都正确注册和注入。测试 DI 容器配置有助于及早发现配置错误，防止可能仅在运行时出现的问题。

为了确认 IoC 容器设置正确，让我们创建一个集成测试，加载实际的模块配置，并验证所有提供者是否已正确注册和注入。

```typescript
import { Test, TestingModule } from '@nestjs/testing';
import { CatsModule } from './cats.module';
import { CatsService } from './cats.service';

describe('Cats Module Integration Test', () => {
  let moduleRef: TestingModule;

  beforeAll(async () => {
    moduleRef = await Test.createTestingModule({
      imports: [CatsModule],
    }).compile();
  });

  it('should resolve exported providers from the ioc container', () => {
    const catsService = moduleRef.get(CatsService);
    expect(catsService).toBeDefined();
  });
});
```

#### 孤立测试、社交测试、集成测试和端到端测试的比较

#### 孤立单元测试

- **重点**：在完全隔离的环境中测试单个单元（类）。
- **使用场景**：测试 `CatsHttpService`。
- **工具**：Suites 的 `TestBed.solitary()` 方法。
- **示例**：模拟 `HttpClient` 并测试 `CatsHttpService`。

#### 社交单元测试

- **重点**：验证单元之间的交互，同时模拟更深层次的依赖项。
- **使用场景**：使用模拟的 `CatsApiService` 并暴露 `CatsDAL` 来测试 `CatsService`。
- **工具**：Suites 的 `TestBed.sociable()` 方法。
- **示例**：模拟 `Prisma` 并测试 `CatsService`。

#### 集成测试

- **重点**：涉及真实的 I/O 操作和完全配置的模块（IoC 容器）。
- **使用场景**：使用 `HttpModule` 和 `nock` 测试 `CatsApiService`。
- **工具**：Nest 的 `TestingModule`。
- **示例**：测试 `HttpClient` 的实际配置和交互。

#### 端到端测试

- **重点**：在更聚合的层面上覆盖类和模块的交互。
- **使用场景**：从最终用户的角度测试系统的完整行为。
- **工具**：Nest 的 `TestingModule`，`supertest`。
- **示例**：使用 `supertest` 模拟 HTTP 请求来测试 `CatsModule`。

有关设置和运行端到端测试的更多详细信息，请参阅 [NestJS 官方测试指南](https://docs.nestjs.com/fundamentals/testing#end-to-end-testing)。