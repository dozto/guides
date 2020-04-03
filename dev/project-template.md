# 项目模版

为了方便项目的创建，统一项目结构，基础功能，所有 Dozto 和 Pard.js 相关的项目(WEB, Module, Service)都需要使用创建工具创建，同时所有涉及模版固定的工作流程都需要遵守模版中的工作流程（commit，release）。

## 项目创建工具

项目创建使用 npx（npm 包执行工具）来自动执行创建脚本，并更新创建项目的`package.json`信息，同时根据不同的创建工具，也会更新不同创建的项目的信息。

因为目前包管理工具以`yarn` 为主，所以需要使用 `yarn create` 命令来创建项目模版。比如需要创建一个 pardjs 的模块包项目，执行`yarn create pardjs-module [生成项目的名称]`。

### Pard.js 项目(开源项目库)

- [create-pardjs-module](https://github.com/pardjs/create-pardjs-module): 用来创建 npm 的 module 模块代码，代码发布后会发布为@pardjs/[项目名称]的公共包。

- [create-pardjs-service](https://github.com/pardjs/create-pardjs-service): 用来创建公共的 service 代码，代码框架为 nest.js。
- [create-pardjs-web](https://github.com/pardjs/create-pardjs-web): 用来创建公共的 web 代码，代码框架为 nuxt.js。

### Dozto 项目

- [create-dozto-module](https://github.com/dozto/create-dozto-module): 用来创建 npm 的 module 模块代码，代码发布后会发布为@dozto/[项目名称]的私有包。

- [create-dozto-service](https://github.com/dozto/create-dozto-service): 用来创建私有的 service 代码，代码框架为 nest.js。
- [create-dozto-web](https://github.com/dozto/create-dozto-web): 用来创建私有的 web 代码，代码框架为 nuxt.js。

> 目前 dozto 还未开通私有 repo

## 项目模版

以上的创建工具都共享同一套代码模版，代码模版如下

- [template-pardjs-module](https://github.com/pardjs/template-pardjs-module)：创建 npm 的 module 的模版。

- [template-pardjs-service](https://github.com/pardjs/template-pardjs-service)：创建 service 的模版。

- [template-pardjs-web](https://github.com/pardjs/template-pardjs-web)：创建 web 项目的模版。

## 基于模版的工作流程

模版通过 script 提供了以下一些实用方法，用来简化、规范、统一日常需要的一些操作流程。

- `yarn commit`: 代替`git commit`命令，整合了 commit 消息的最佳范式。

- `yarn release`: 发布 npm 的 module，或 docker 镜像。

  > ⚠️ 这里的 release 是会自动升级版本，版本使用语意版本控制，也就是`MAJOR.MINOR.PATCH`，需要注意的是`MAJOR`代表向后不兼容的更新，`MINOR`代表向后兼容，但是有增强的更新，`PATCH`代表向后兼容的错误修复更新。

- `yarn build`: 构建项目，在 module 项目中为构建编译后的 lib 文件，在 web 项目中为构建静态所需的 js 和 css 文件，service 项目中为构建编译后的 js 文件。

- `yarn build:doc`：构建文档，基于 ts 特性自动构建 module 文档（仅在 module 项目中使用）。

- `yarn generate`: 生成静态 html 页面（仅在 web 项目中使用）。

- `yarn test`：执行项目的测试代码，包涵代码覆盖率测试。

- `yarn test:watch`：执行监视模式的代码测试。

- `yarn dev`: 在开发模式启动项目。(开发中使用这个命令启动项目)

- `yarn start`: 再生产环境模式启动项目。（生产环境中使用这个命令启动项目）

## 错误处理

### 返回数据解构
```yaml
meta: object! 
data: any! # 使用object或array返回数据，不需要返回数据的地方返回`null` (创建返回新建的数据，更新返回更新后的数据，删除返回空)
error: object
```

### Meta包含字段
```yaml
status: string! # 记录错误代码，用户graphql或restful返回
previous: string # Cursor 分页 上一页
next: string # Cursor 分页 下一页
hasNext: boolean # Cursor 分页 是否还有下一页
```

### Error包含字段
```yaml
service: string! # 记录错误发生所在的服务
code: string! # 用于开发之间交流沟通错误，从开发角使用有意义的内容如`USER_INVALID_ROLE`
message: string! # 用于对用户说明错误内容，需要做i18n根据`accept-language`
traceId: string! # 用户追踪请求的id，请求id来自`X-Trace-Id`，可以用来查询log来查询相关请求。
isSentry: bool! # 记录错误是否上报到sentry。
exceptions: [Errors] # 导致问题的其他Error，来自其他service，或第三方服务，或者是校验错误。
```
