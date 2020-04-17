# 🧩 Git Flow

为了方便代码管理以及 CI/CD 集成需要，在开发中我们遵循 Vincent Driessen 提出的 [Git Flow 流程](https://nvie.com/posts/a-successful-git-branching-model/)，并需要确保：

- `master`分支：保留发布到生产环境中的最新代码并标记`tag`，所有合并到 master 分支的代码都会**自动发布到生产(prod)环境中**。
- `develop`分支：保留可部署的最新代码，所有合并到`develop`分支的代码都会**自动发布到开发(dev)和演示(stag)环境中**。

![image.png](https://storage.googleapis.com/slite-api-files-production/files/c83b048e-2a83-45a2-8a3d-8c76caa0d36e/image.png)

## 使用工具

- `git-flow-avh` ：`brew install git-flow-avh` 安装 [gitflow-avh](https://github.com/petervanderdoes/gitflow-avh) 到系统中。 使用说明参考 [git-flow cheatsheet](https://danielkummer.github.io/git-flow-cheatsheet/index.html) 内容。

> 功能分支完成后需要使用`git flow feature publish` 将分支代码提交到仓库，并从 Github 发起分支到 `develop` 的 PR 请求。  
> 仅在个人维护且不需要代码 Review 的情况下可以使用`git flow feature finish` 。

# 🐞 错误处理

- 每个项目服务端需要一个最终的错误处理，所有未处理的异常都会由这个错误处理。(知道来自哪个 service)
- 项目中每个业务功能都需要一个通用的错误处理，在功能模块发生的未处理的异常都会有这个错误处理。(知道来自哪个功能)
- 针对第三方服务或系统进行单独的错误封装，这样在第三方服务或系统发生错误的来源和信息(知道错误和我们无关)
- 项目中需要异常处理的地方，都有自己定义的错误类型，当这个功能发生错误时按照需求的说明反对可以对应的错误。(前端根据特定错误进行特定处理)

## 面向错误开发

在开发是优先考虑错误处理情况，在做涉及到请求处理时，先使用通用的异常处理捕获异常

- 在前端开发中根据项目样式添加针对错误的弹窗
- 后台在开发中自动适配错误的消息通知。
- 后台根据上游错误自动记录日志/上报 sentry

## 错误可复现

为了解决开发中出现的各种异常，同时考虑到客户一般不会有能力提供复现方法，需要尽可能的记录异常情况，方便再出现异常时定位错误

> 在 Response 中错误需要显示的内容参考下面 Response 结构部分。

# 📥 Request&Response

## 请求可追踪(通用库实现)

为每一条请求添加一个唯一的`X-Trace-Id`, 对异常的错误记录日志。(**日志管理系统**)

### 返回数据解构

```yaml
meta: object!
data: [object || array || null]! # 使用object或array返回数据，不需要返回数据的地方返回`null` (创建返回新建的数据，更新返回更新后的数据，删除返回空)
error: object
```

### Meta 包含字段

Meta 信息在所有返回中都需要包含。

```yaml
status: string! # 记录错误代码，用户graphql或restful返回
previous: string # Cursor 分页 上一页
next: string # Cursor 分页 下一页
hasNext: boolean # Cursor 分页 是否还有下一页
```

### Error 包含字段

错误仅需要在服务端发生错误的时候返回。

```yaml
service: string! # 记录错误发生所在的服务
code: string! # 用于开发之间交流沟通错误，从开发角使用有意义的内容如`USER_INVALID_ROLE`
message: string! # 用于对用户说明错误内容，需要做i18n根据`accept-language`
exceptions: [Errors] # 导致问题的其他Error，来自其他service，或第三方服务，或者是校验错误。
traceId: string! # 用户追踪请求的id，请求id来自`X-Trace-Id`，可以用来查询log来查询相关请求。
isSentry: bool! # 记录错误是否上报到sentry。
```

## 使用通用库，脚手架

为了较少不必要的重复，同时提高开发效率，遵守 DRY 原则，定期将常用的代码和功能做成通用组件包，并上传到开发模版中，

在开发前应检查有哪些已有的工具包，并在开发中优先使用这些通用组件包中的内容或方法。

## 核心代码的测试

根据不同项目和客户的要求，对测试要请也不一样，不过对于项目核心功能(核心业务逻辑，支付等)最少提供单元测试代码(使用 jest)。

同时抽象出来共享的组件包需要有 99%+的测试覆盖率保证功能正常。
