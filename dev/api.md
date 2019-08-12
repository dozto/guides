# HTTP API 设计指南

> 翻译自 `HTTP API Design Guide` [https://github.com/interagent/http-api-design](https://github.com/interagent/http-api-design)
>
> 由 Dozto 根据自身实践改进，最后更新日期**12th Aug, 2019**

## 目录

- 基础
  - 强制使用安全连接（Secure Connections）
  - 强制头信息 Accept 中提供版本号
  - 为内省而提供 X-Request-Id
  - 在Header或Cookie信息中的包含认证信息*
  - 为多语言客户端通过`Accept-Language`提供多语言支持*
  - 访问系统的关键功能需要提供管理员的X-Admin-Key*
  
- 请求（Requests）
  - 在请求的 body 体使用 JSON 格式数据
  - 使用统一的资源路径格式
  - 路径和属性要小写
  - 支持方便的无 id 间接引用
  - 最小化路径嵌套
  
- 响应（Responses）
  - 返回合适的状态码
  - 提供全部可用的资源
  - 提供资源的(UU)ID
  - 提供标准的时间戳
  - 使用 UTC（世界标准时间）时间，用 ISO8601 进行格式化
  - 嵌套外键关系
  - 生成结构化的错误*
  
- 工件（Artifacts）
  
  * 提供服务的状态查询接口*
  
  - 提供机器可读的 JSON 模式
  - 提供人类可读的文档
  - 提供可执行的例子
  - 描述稳定性

### 基础

#### 隔离关注点

设计时通过将请求和响应之间的不同部分隔离来让事情变得简单。保持简单的规则让我们能更关注在一些更大的更困难的问题上。

请求和响应将解决一个特定的资源或集合。使用路径（path）来表明身份，body 来传输内容（content）还有头信息（header）来传递元数据（metadata）。查询参数同样可以用来传递头信息的内容，但头信息是首选，因为他们更灵活、更能传达不同的信息。

#### 强制使用安全连接（Secure Connections）

所有的访问 API 行为，都需要用 TLS 通过安全连接来访问。没有必要搞清或解释什么情况需要 TLS 什么情况不需要 TLS，直接强制任何访问都要通过 TLS。

理想状态下，通过拒绝所有非 TLS 请求，不响应 http 或 80 端口的请求以避免任何不安全的数据交换。如果现实情况中无法这样做，可以返回`403 Forbidden`响应。

把非 TLS 的请求重定向(Redirect)至 TLS 连接是不明智的，这种含混/不好的客户端行为不会带来明显好处。依赖于重定向的客户端访问不仅会导致双倍的服务器负载，还会使 TLS 加密失去意义，因为在首次非 TLS 调用时，敏感信息就已经暴露出去了。

#### 强制头信息 Accept 中提供版本号

制定版本并在版本之间平缓过渡对于设计和维护一套 API 是个巨大的挑战。所以，最好在设计之初就使用一些方法来预防可能会遇到的问题。

为了避免 API 的变动导致用户使用中产生意外结果或调用失败，最好强制要求所有访问都需要指定版本号。请避免提供默认版本号，一旦提供，日后想要修改它会相当困难。

最适合放置版本号的位置是头信息(HTTP Headers)，在 `Accept` 段中使用自定义类型(content type)与其他元数据(metadata)一起提交。例如:

```
Accept: application/vnd.heroku+json; version=3
```

#### 为内省而提供 X-Request-Id

为每一个请求响应包含一个`X-Request-Id`头，并使用 UUID 作为该值。通过在客户端、服务器或任何支持服务上记录该值，它能为我们提供一种机制来跟踪、诊断和调试请求。由 Gateway 来处理。

#### 在Header或Cookie信息中的包含认证信息

在需要身份认证的服务器中，对web项目可以使用Cookie信息来提供身份认证信息，如`Cookie: authentication=xxxx;` 对APP等可以使用`authentication` header来提供身份验证信息。

后端服务需要同时支持两种形式的身份验证，方便不同客户端对API进行调用。 

#### 为多语言客户端端通过`Accept-Language`提供多语言支持

在需要多语言支持的情况下，后端返回的人类可读信息需要匹配请求中的`Accpet-Language`，可用的`Accpet-Language`值包括`zh`, `en`, `zh-CN`以及 `en-US`。这里的`Accept-Language`是一个可选选项，默认值根据不同项目而不同。

例如，在`Accpet-Language` 为`zh`的情况下，返回的错误信息如下。

```json
{
  "errors": [{
    "requestId": "01234567-89ab-cdef-0123-456789abcdef"
    "type": "RATE_LIMIT_RACHED",
    "message": "API反问达到上限，请稍后再试。",
    "status": "429",
    "isSentry": true
  }]
}
```

#### 访问系统的关键功能需要提供管理员的X-Admin-Key

通常在系统中会有一些仅供管理员查看/调用的API，对于这些API需要提供特殊的管理员可以访问的`X-Admin-Key`来进行身份验证。

比如，每个service中可能会有的config相关的API，和seting相关的API,以及查看服务器状态和API文档的请求都需要校验这个key，这个key永远不存储在客户端中，仅供开发者/管理员手动管理。

### 请求（Requests）

#### 在请求的 body 体使用 JSON 格式数据

在 `PUT`/`PATCH`/`POST` 请求的正文（request bodies）中使用 JSON 格式数据，而不是使用 form 表单形式的数据。这与我们使用 JSON 格式返回请求相对应，例如:

```bash
$ curl -X POST https://service.com/apps \
    -H "Content-Type: application/json" \
    -d '{"name": "demoapp"}'
{
  "id": "01234567-89ab-cdef-0123-456789abcdef",
  "name": "demoapp",
  "owner": {
    "email": "username@example.com",
    "id": "01234567-89ab-cdef-0123-456789abcdef"
  },
  ...
}
```

#### 使用统一的资源路径格式

##### 资源名（Resource names）

使用**复数形式**为资源命名，除非这个资源在系统中是*单例*的 (例如，在大多数系统中，给定的用户帐户只有一个)。 这种方式保持了特定资源的统一性。

##### 行为（Actions）

好的末尾不需要为资源指定特殊的行为，但在特殊情况下，为某些资源指定行为却是必要的。为了描述清楚，在行为前加上一个标准的`actions`：

```
/resources/:resource/actions/:action
```

例如：

```
/runs/{rund}/actions/stop
```

#### 路径和属性要小写

为了和域名命名规则保持一致，使用小写字母并用`-`分割路径名字，例如：

```
service-api.com/users
service-api.com/app-setups
```

属性也使用驼峰命名方式。 例如：

```json
serviceClass: "first"
```

#### 支持方便的无 id 间接引用

在某些情况下，让用户提供 ID 去定位资源是不方便的。例如，一个用户想取得他在 Heroku 平台 app 信息，但是这个 app 的唯一标识是 UUID。这种情况下，你应该支持接口通过名字和 ID 都能访问，例如:

```bash
$ curl https://service.com/apps/{appIdOrName}
$ curl https://service.com/apps/97addcf0-c182
$ curl https://service.com/apps/www-prod
```

不要只接受使用名字而放弃了使用 id。

#### 最小化路径嵌套

在一些有父路径/子路径嵌套关系的资源数据模块中，路径可能有非常深的嵌套关系，例如:

```
/orgs/{orgId}/apps/{appId}/dynos/{dynoId}
```

推荐在根(root)路径下指定资源来限制路径的嵌套深度。使用嵌套指定范围的资源。在上述例子中，dyno 属于 app，app 属于 org 可以表示为：

```
/orgs/{orgId}
/orgs/{orgId}/apps
/apps/{appId}
/apps/{appId}/dynos
/dynos/{dynoId}
```

### 响应（Responses）

#### 返回合适的状态码

为每一次的响应返回合适的 HTTP 状态码。 好的响应应该使用如下的状态码:

- `200`: `GET`请求成功，及`DELETE`或`PATCH`同步请求完成，或者`PUT`同步更新一个已存在的资源
- `201`: `POST` 同步请求完成，或者`PUT`同步创建一个新的资源
- `202`: `POST`，`PUT`，`DELETE`，或`PATCH`请求接收，将被异步处理

使用身份认证（authentication）和授权（authorization）错误码时需要注意：

- `401 Unauthorized`: 用户未认证，请求失败
- `403 Forbidden`: 用户无权限访问该资源，请求失败

当用户请求错误时，提供合适的状态码可以提供额外的信息：

- `400 Bad Request`: 请求的业务被拒绝
- `404 Not Found`: 请求的资源不存在
- `429 Too Many Requests`: 因为访问频繁，你已经被限制访问，稍后重试
- 500 Internal Server Error`: 服务器错误，确认状态并报告问题

对于用户错误和服务器错误情况状态码，参考： [HTTP response code spec](https://tools.ietf.org/html/rfc7231#section-6)

#### 提供全部可用的资源

提供全部可显现的资源表述 (例如： 这个对象的所有属性) ，当响应码为 200 或是 201 时返回所有可用资源，包含 `PUT`/`PATCH` 和 `DELETE`
请求，例如:

```json
$ curl -X DELETE \
  https://service.com/apps/1f9b/domains/0fd4

HTTP/1.1 200 OK
Content-Type: application/json;charset=utf-8
...
{
  "createdAt": "2012-01-01T12:00:00Z",
  "hostname": "subdomain.example.com",
  "id": "01234567-89ab-cdef-0123-456789abcdef",
  "updatedAt": "2012-01-01T12:00:00Z"
}
```

当请求状态码为 202 时，不返回所有可用资源，例如：

```bash
$ curl -X DELETE \
  https://service.com/apps/1f9b/dynos/05bd

HTTP/1.1 202 Accepted
Content-Type: application/json;charset=utf-8
...
{}
```

#### 提供资源的(UU)ID

在默认情况给每一个资源一个`id`属性。除非有更好的理由，否则请使用**UUID**。不要使用那种在服务器上或是资源中不是全局唯一的标识，尤其是自动增长的 id。

生成小写的 UUID 格式 `8-4-4-4-12`，例如：

```json
"id": "01234567-89ab-cdef-0123-456789abcdef"
```

#### 提供标准的时间戳

为资源提供默认的创建时间 `createdAat` 和更新时间 `updatedAt`，例如:

```json
{
  ...
  "createdAt": "2012-01-01T12:00:00Z",
  "updatedAt": "2012-01-01T13:00:00Z",
  ...
}
```

有些资源不需要使用时间戳那么就忽略这两个字段。

#### 使用 UTC（世界标准时间）时间，用 ISO8601 进行格式化

仅接受和返回 UTC 格式的时间。ISO8601 格式的数据，例如:

```json
"finishedAt": "2012-01-01T12:00:00Z"
```

#### 嵌套外键关系

使用嵌套对象序列化外键关联，例如:

```json
{
  "name": "service-production",
  "owner": {
    "id": "5d8201b0..."
  }
  // ...
}
```

而不是像这样:

```json
{
  "name": "service-production",
  "ownerId": "5d8201b0...",
  ...
}
```

这种方式尽可能的把相关联的资源信息内联在一起，而不用改变资源的结构，或者引入更多的顶层字段，例如:

```json
{
  "name": "service-production",
  "owner": {
    "id": "5d8201b0...",
    "name": "Alice",
    "email": "alice@heroku.com"
  },
  ...
}
```

#### 生成结构化的错误

响应错误的时，生成统一的、结构化的错误信息。所有错误需要放在`errors`中返回给前端，同时每个error需要遵循以下结构。

```yaml
- requestId: 请求的X-Request-Id，方便对异常进行追踪。
- type: 错误类型，为英文的错误定义名称，如`RATE_LIMIT_RACHED`, `DUPLICATED_EMAIL`等。
- serviceType: 服务类型，通过服务类定定义错误发生所在的服务，如`USER`, `FORM`, `WECHAT`等。
- message: 错误需要提供的通知消息，供客户端在客户端展示，是人类客户的友好的错误信息，需要支持多语言。
- isSentry: 错误是否发送到sentry。
- throwedAt: 错误发生的时间。
```

```
HTTP/1.1 429 Too Many Requests
```

```json
{
  "errors": [{
    "requestId": "01234567-89ab-cdef-0123-456789abcdef"
    "type": "RATE_LIMIT_RACHED",
    "serviceType": "USER"
    "message": "Account reached its API rate limit.",
    "isSentry": true,
    "throwedAt": "2012-01-01T12:00:00Z"
  }]
}
```

同时自动生成错误文档，以及客户端可能遇到的错误信息`type`。

请求中多余的空格会增加响应大小，而且现在很多的 HTTP 客户端都会自己输出可读格式（"prettify"）的 JSON。所以最好保证响应 JSON 最小化，例如：

```json
{
  "beta": false,
  "email": "alice@heroku.com",
  "id": "01234567-89ab-cdef-0123-456789abcdef",
  "lastLogin": "2012-01-01T12:00:00Z",
  "createdAt": "2012-01-01T12:00:00Z",
  "updatedAt": "2012-01-01T12:00:00Z"
}
```

而不是这样：

```json
{
  "beta": false,
  "email": "alice@heroku.com",
  "id": "01234567-89ab-cdef-0123-456789abcdef",
  "lastLogin": "2012-01-01T12:00:00Z",
  "createdAt": "2012-01-01T12:00:00Z",
  "updatedAt": "2012-01-01T12:00:00Z"
}
```

你可以提供可选的方式为客户端提供更详细可读的响应，使用查询参数（例如：`?pretty=true`）或者通过`Accept`头信息参数（例如：`Accept: application/vnd.heroku+json; version=3; indent=4;`）。

### 工件（Artifacts）

#### 提供服务的状态查询接口

所有的服务均提供`/api/status` 来返回当前服务的状态。在每个状态返回结果中需要饱含一下内容：

```yaml
- 
```

#### 提供完整的 Swagger 文档

所有的 API 需要提供完整的请求参数，相应的响应 Model 信息和 Swagger 文档。并准确表明所对应的 API 版本信息。

#### 提供人类可读的文档

提供人类可读的文档让客户端开发人员可以理解你的 API。

参见[nest Documentation](https://docs.nestjs.com/recipes/documentation)

除了节点信息，提供一个 API 概述信息:

- 验证授权，包含如何取得和如何使用 token。
- API 稳定及版本管理，包含如何选择所需要的版本。
- 一般情况下的请求和响应的头信息。
- 错误的序列化格式。

#### 描述稳定性

描述每个的 API 的稳定性或是它在各种各样节点环境中的完备性和稳定性，例如：加上 原型版（prototype）/开发版（development）/产品版（production）等标记（默认省略）。

更多关于可能的稳定性和改变管理的方式，查看 [Heroku API compatibility policy](https://devcenter.heroku.com/articles/api-compatibility-policy)

一旦你的 API 宣布产品正式版本及稳定版本时，不要在当前 API 版本中做一些不兼容的改变。如果你需要，请创建一个新的版本的 API。
