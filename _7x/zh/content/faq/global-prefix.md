### 全局前缀

要为HTTP应用中注册的**每个路由**设置前缀，请使用`INestApplication`实例的`setGlobalPrefix()`方法。

```typescript
const app = await NestFactory.create(AppModule);
app.setGlobalPrefix('v1');
```

你可以使用以下结构将路由从全局前缀中排除：

```typescript
app.setGlobalPrefix('v1', {
  exclude: [{ path: 'health', method: RequestMethod.GET }],
});
```

或者，你可以将路由指定为字符串（这将适用于所有请求方法）：

```typescript
app.setGlobalPrefix('v1', { exclude: ['cats'] });
```

> info **提示** `path`属性支持使用[path-to-regexp](https://github.com/pillarjs/path-to-regexp#parameters)包的通配符参数。注意：这不接受通配符星号`*`。相反，你必须使用参数（例如，`(.*)`，`:splat*`）。