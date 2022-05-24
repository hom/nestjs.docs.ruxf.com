### 接口版本

> info **Hint** 本章节仅适用于以 HTTP 构建的应用程序.

接口版本可以在同一个应用程序中的控制器或者路由层面支持 **不同的版本**。 应用程序经常更改，在仍然需要支持以前版本的应用程序的同时，需要进行重大更改的情况并不少见。

Nest.js 支持一下 4 种形式的版本管理：

<table>
  <tr>
    <td><a href='techniques/versioning#uri-versioning-type'><code>URI 版本类型</code></a></td>
    <td>版本在请求的 URI 中传递 （默认）</td>
  </tr>
  <tr>
    <td><a href='techniques/versioning#header-versioning-type'><code>Header 版本管理</code></a></td>
    <td>在自定义的 Header 中传递版本</td>
  </tr>
  <tr>
    <td><a href='techniques/versioning#media-type-versioning-type'><code>Media Type 版本管理</code></a></td>
    <td>在 <code>Accept</code> 头部标签中声明版本</td>
  </tr>
  <tr>
    <td><a href='techniques/versioning#custom-versioning-type'><code>自定义版本管理</code></a></td>
    <td>请求的任何部分都可用于指定版本，需要提供一个自定义函数来提取所述版本</td>
  </tr>
</table>

#### URI 版本类型

URI 版本管理在请求地址中标识版本，比如 `https://example.com/v1/route` 和 `https://example.com/v2/route`。

> warning **Notice** URI 版本管理会在 <a href="faq/global-prefix">全局路径前缀</a> （如果存在）自动添加版本号， 并且在控制器或路由之前.

按照以下操作来使用 URI 版本管理:

```typescript
@@filename(main)
const app = await NestFactory.create(AppModule);
// or "app.enableVersioning()"
app.enableVersioning({
  type: VersioningType.URI,
});
await app.listen(3000);
```

> warning **Notice** URI 版本管理默认使用 `v` 前缀， 并且可以通过设置 `prefix` 来自定义前缀或者设置 `false` 来取消使用前缀行为。

> info **Hint** `VersioningType` 中的类型 `type` 是从 `@nestjs/common` 引入的枚举。

#### Header 版本类型

Header 版本管理使用用户指定的自定义头部标签来标明使用的请求版本。

使用 Header 版本管理的示例 HTTP 请求

按照以下步骤为你的应用程序启用 **Header 版本管理**。

```typescript
@@filename(main)
const app = await NestFactory.create(AppModule);
app.enableVersioning({
  type: VersioningType.HEADER,
  header: 'Custom-Header',
});
await app.listen(3000);
```

`header` 属性标明要用来传递接口版本的头部标签名。

> info **Hint** `VersioningType` 中的类型 `type` 是从 `@nestjs/common` 引入的枚举。

#### Media Type 版本类型

Media 版本类型使用 `Accept` 头部标签来声明请求的版本。

在 `Accept` 头部标签内， 版本将与媒体类型用分号`;`分隔。它应该包含一个键值对，表示要用于请求的版本，例如 `Accept： application/json;v=2`。在配置包含键和分隔符的版本时，可以设置 `key` 的值作为键的前缀值。

要为应用程序启用 **媒体类型版本控制**，请执行以下操作：

```typescript
@@filename(main)
const app = await NestFactory.create(AppModule);
app.enableVersioning({
  type: VersioningType.MEDIA_TYPE,
  key: 'v=',
});
await app.listen(3000);
```

`key` 属性应该是包含该版本的键值对的键和分隔符。在示例中 `Accept： application/json;v=2`，`key` 属性将设置为 `v=`。

> info **Hint** `版本控制类型` 枚举可用于 `type` 属性，并从 `@nestjs/common` 包中导入。

#### 自定义版本类型

自定义版本控制使用请求的任何部分来指定一个或多个版本，并使用返回字符串或字符串数组的 `extractor` 函数分析传入请求。

如果请求者提供了多个版本，则提取器函数 `extractor` 可以返回一个字符串数组，这些字符串按最大、最高版本到最小、最低版本的顺序排序，版本控制按从高到低的顺序与路由匹配。

如果从 `extractor` 返回空字符串或数组，则不会匹配任何路由，并返回 `404`。

例如，如果传入请求指定它支持版本 `1`、`2` 和 `3`，则 `提取器` **必须** 返回 `[3， 2， 1]`，这可确保首先选择最可能的最高路由版本。

如果提取了版本 `[3， 2， 1]`，但仅存在版本 `2` 和 `1` 的路由，则选择与版本 `2` 匹配的路由（自动忽略版本 `3`）。

> warning **Notice** 由于设计限制，根据从提取器 `extractor` 中返回的数组中选择最高匹配版本 **不能可靠地** 与 `Express` 适配器使用，单个版本（1 个字符串或 1 个元素的数组）在 `Express` 中工作正常，而`Fastify` 能正确支持最高匹配版本选择和单个版本选择。

要为应用程序启用 **自定义版本控制**，请创建一个 `extractor` 函数并将其传递到应用程序中，如下所示：

```typescript
@@filename(main)
// Example extractor that pulls out a list of versions from a custom header and turns it into a sorted array.
// This example uses Fastify, but Express requests can be processed in a similar way.
const extractor = (request: FastifyRequest): string | string[] =>
  [request.headers['custom-versioning-field'] ?? '']
     .flatMap(v => v.split(','))
     .filter(v => !!v)
     .sort()
     .reverse()

const app = await NestFactory.create(AppModule);
app.enableVersioning({
  type: VersioningType.CUSTOM,
  extractor,
});
await app.listen(3000);
```

#### 示例

版本控制允许您对控制器、各个路由进行版本控制，并且还为某些资源提供了一种选择退出版本控制的方法。无论应用程序使用哪种版本控制类型，版本控制的用法都是相同的。

> warning **Notice** 如果为应用程序启用了版本控制，但控制器或路由未指定版本，则对该控制器、路由的任何请求都将返回 `404` 响应状态。同样，如果收到的请求包含没有相应控制器或路由的版本，则还将返回 `404` 响应状态。

#### 控制器版本

可以在独立的控制器中指定版本，此设置将影响控制器下的所有路由。

参照以下添加针对单个路由的版本：

```typescript
@@filename(cats.controller)
@Controller({
  version: '1',
})
export class CatsControllerV1 {
  @Get('cats')
  findAll(): string {
    return 'This action returns all cats for version 1';
  }
}
@@switch
@Controller({
  version: '1',
})
export class CatsControllerV1 {
  @Get('cats')
  findAll() {
    return 'This action returns all cats for version 1';
  }
}
```

#### 路由版本

可以在独立的路由中指定版本，此版本将覆盖其他影响路由的版本控制比如设置的控制器版本。

参照以下添加针对单个路由的版本：

```typescript
@@filename(cats.controller)
import { Controller, Get, Version } from '@nestjs/common';

@Controller()
export class CatsController {
  @Version('1')
  @Get('cats')
  findAllV1(): string {
    return 'This action returns all cats for version 1';
  }

  @Version('2')
  @Get('cats')
  findAllV2(): string {
    return 'This action returns all cats for version 2';
  }
}
@@switch
import { Controller, Get, Version } from '@nestjs/common';

@Controller()
export class CatsController {
  @Version('1')
  @Get('cats')
  findAllV1() {
    return 'This action returns all cats for version 1';
  }

  @Version('2')
  @Get('cats')
  findAllV2() {
    return 'This action returns all cats for version 2';
  }
}
```

#### 多版本

可以在控制器或路由中设置支持多版本，你需要以数组的形式设置多版本支持。

参照以下设置多版本支持

```typescript
@@filename(cats.controller)
@Controller({
  version: ['1', '2'],
})
export class CatsController {
  @Get('cats')
  findAll(): string {
    return 'This action returns all cats for version 1 or 2';
  }
}
@@switch
@Controller({
  version: ['1', '2'],
})
export class CatsController {
  @Get('cats')
  findAll() {
    return 'This action returns all cats for version 1 or 2';
  }
}
```

#### 无影响的路由

有的控制器或路由不关心接口版本并且无论版本如何，都将具有相同的功能，为了适应这种情况，可以将版本设置为 `VERSION_NEUTRAL`。

传入的请求将映射到 `VERSION_NEUTRAL` 控制器或路由，而不管请求中发送的版本如何，或者请求中根本不包含版本信息。

> warning **Notice** 对于 URI 版本控制，`VERSION_NEUTRAL` 不会在 URI 中指定版本信息。

要添加非特定版本控制器或路由，请执行以下操作：

```typescript
@@filename(cats.controller)
import { Controller, Get, VERSION_NEUTRAL } from '@nestjs/common';

@Controller({
  version: VERSION_NEUTRAL,
})
export class CatsController {
  @Get('cats')
  findAll(): string {
    return 'This action returns all cats regardless of version';
  }
}
@@switch
import { Controller, Get, VERSION_NEUTRAL } from '@nestjs/common';

@Controller({
  version: VERSION_NEUTRAL,
})
export class CatsController {
  @Get('cats')
  findAll() {
    return 'This action returns all cats regardless of version';
  }
}
```

#### 全局默认版本

如果你不想为每一个控制器或路由指定版本，或者你想为每一个控制器、路由指定默认的版本而不指定具体的版本号，你可以参照以下设置 `defaultVersion`：

```typescript
@@filename(main)
app.enableVersioning({
  // ...
  defaultVersion: '1'
  // or
  defaultVersion: ['1', '2']
  // or
  defaultVersion: VERSION_NEUTRAL
});
```
