### 迁移指南

如果您当前正在使用 `@nestjs/swagger@3.*`，请注意 4.0 版本中的以下重大变更/API 更改。

#### 重大变更

以下装饰器已被更改/重命名：

- `@ApiModelProperty` 现在是 `@ApiProperty`
- `@ApiModelPropertyOptional` 现在是 `@ApiPropertyOptional`
- `@ApiResponseModelProperty` 现在是 `@ApiResponseProperty`
- `@ApiImplicitQuery` 现在是 `@ApiQuery`
- `@ApiImplicitParam` 现在是 `@ApiParam`
- `@ApiImplicitBody` 现在是 `@ApiBody`
- `@ApiImplicitHeader` 现在是 `@ApiHeader`
- `@ApiOperation({{ '{' }} title: 'test' {{ '}' }})` 现在是 `@ApiOperation({{ '{' }} summary: 'test' {{ '}' }})`
- `@ApiUseTags` 现在是 `@ApiTags`

`DocumentBuilder` 重大变更（更新了的方法签名）：

- `addTag`
- `addBearerAuth`
- `addOAuth2`
- `setContactEmail` 现在是 `setContact`
- `setHost` 已被移除
- `setSchemes` 已被移除（请改用 `addServer`，例如：`addServer('http://')`）

#### 新增方法

新增了以下方法：

- `addServer`
- `addApiKey`
- `addBasicAuth`
- `addSecurity`
- `addSecurityRequirements`

<app-banner-devtools></app-banner-devtools>