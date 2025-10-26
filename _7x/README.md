# 7x nestjs 文档 

这个文档站点是使用 nestjs 作为服务器的，其不需要构架成 html 文件

## 文档服务启动
在 `_7x/zh` 目录下执行 `npm run start ` 即可

执行 `npm run build` 命令生成静态文件,输出在 dist 目录下

## prompt

```
你是一个 nestjs 技术专家，你需要做 nestjs 的文档翻译工作，帮我把下面的 nestjs 文档翻译成中文，要求：
- 保持 markdown 的格式不变，
- 不用翻译代码块
- 不用翻译注释内容
- 不要翻译代码块中的注释内容
```

## 菜单
在 `_7x/zh/src/app/homepage/menu/menu.component.ts` 配置菜单

## 文档翻译
在 `_7x/zh/src/app/homepage/homepage.component.ts` 找到r如下代码，需要申请 algoliaApiKey

``` ts
  createDocSearchScriptTag(): HTMLScriptElement {
    const scriptTag = document.createElement('script');
    scriptTag.type = 'text/javascript';
    scriptTag.src = 'https://cdn.jsdelivr.net/npm/@docsearch/js@3';
    scriptTag.async = true;
    scriptTag.onload = () => {
      (window as any).docsearch({
        apiKey: environment.algoliaApiKey,
        indexName: 'nestjs',
        container: '#search',
        appId: 'SDCBYAN96J',
        debug: false,
      });
    };
    return scriptTag;
  }
  ```


## 代码提交

使用 git commit --no-verify -m "xxx" 提交代码,忽略检测