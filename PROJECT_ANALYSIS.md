# Server 仓库分析（ONLYOFFICE Document Server 后端）

## 1) 项目是做什么的

该仓库是 ONLYOFFICE Document Server 的后端服务层，负责：

- 文档协同编辑服务（DocService）
- 文档格式转换服务（FileConverter）
- 拼写检查服务（SpellChecker）
- 指标采集（Metrics / StatsD）
- 公共能力层（Common）：配置、日志、存储、消息队列、租户上下文、通知等

定位上，它不是单体的“一个 Node 进程”，而是**多个 Node 子服务组成的后端服务集合**。

---

## 2) 整体架构与技术栈

### 2.1 架构分层

- **对外主服务（DocService）**
  - Express HTTP 服务 + Socket.IO 实时通道
  - 负责：协同编辑会话、文档变更、上传下载、转换接口接入、健康检查等
- **异步转换子服务（FileConverter）**
  - 基于消息队列的任务消费与转换执行
  - 通过 Node `cluster` 进行 worker 扩展
- **拼写子服务（SpellChecker）**
  - 独立 Node 服务，负责拼写建议
  - 同样使用 `cluster` 做 master/worker 与健康检查
- **基础设施层（Common）**
  - 统一配置（`node-config`）、日志（log4js）、存储（FS/S3/Azure）、队列（RabbitMQ/ActiveMQ）、数据库连接器等

### 2.2 关键技术栈

- 语言/运行时：Node.js
- Web：Express
- 实时协同：Socket.IO（另有 SockJS 能力）
- 并发模型：Node `cluster`（用于 FileConverter、SpellChecker）
- 队列：RabbitMQ（默认）/ ActiveMQ
- 数据层：PostgreSQL（默认）+ MySQL/MSSQL/Oracle/Dameng
- 缓存：Redis
- 存储：本地 FS / Amazon S3 / Azure Blob
- 测试：Jest（unit/integration/perf）

---

## 3) 核心入口文件与主要模块

### 3.1 入口文件

- **DocService 主入口**：`DocService/sources/server.js`
- **协同核心模块**：`DocService/sources/DocsCoServer.js`
- **FileConverter 入口（master/worker）**：`FileConverter/sources/convertermaster.js`
- **FileConverter 执行核心**：`FileConverter/sources/converter.js`
- **SpellChecker 入口**：`SpellChecker/sources/server.js`

### 3.2 主要模块（按职责）

- **DocService**
  - `DocsCoServer.js`：协同连接、会话/锁、变更处理、任务交互
  - `converterservice.js`：转换链路入口
  - `canvasservice.js`：下载/打印/保存等文档服务
  - `routes/*`：静态与信息路由
  - `databaseConnectors/*`：多数据库连接器
- **FileConverter**
  - `convertermaster.js`：worker 生命周期管理
  - `converter.js`：拉取任务、下载文件、执行转换、签名流程
  - `signing/*`：PDF 签名（AWS KMS / CSC）
- **Common**
  - `storage/*`：FS/S3/Azure 抽象
  - `taskqueueRabbitMQ.js`、`rabbitMQCore.js`：消息队列抽象
  - `operationContext.js`、`tenantManager.js`：请求/租户上下文

---

## 4) 开发环境如何启动、调试

> 仓库内有多包结构（root + Common + DocService + FileConverter + SpellChecker + Metrics）。建议先安装各子包依赖，再按角色启动服务。

### 4.1 安装依赖

在仓库根目录执行：

```bash
npm run build
```

该命令会并行执行 `install:*`，对 `Common/DocService/FileConverter/Metrics` 分别运行 `npm ci`。

### 4.2 启动主服务（DocService）用于开发调试

Linux/macOS 常见方式（示例）：

```bash
cd DocService
NODE_ENV=development-linux NODE_CONFIG_DIR=../Common/config node sources/server.js
```

Windows 可参考仓库说明中的 `run.bat` 思路（历史文档偏 Windows）。

### 4.3 启动转换子服务（FileConverter）

```bash
cd FileConverter
NODE_ENV=development-linux NODE_CONFIG_DIR=../Common/config node sources/convertermaster.js
```

### 4.4 启动拼写子服务（SpellChecker）

```bash
cd SpellChecker
NODE_ENV=development-linux NODE_CONFIG_DIR=../Common/config node sources/server.js
```

### 4.5 调试建议

- 开发配置文件位于 `Common/config/development-*.json`
- 主服务默认端口在 `services.CoAuthoring.server.port`（默认 8000）
- 若要观察协同链路，重点看 DocService 日志；若是转换问题，重点看 FileConverter 日志与队列
- 结合 `NODE_ENV` + `NODE_CONFIG_DIR` 切换环境与配置覆盖

---

## 5) 如何编译/打包

这个仓库既有 npm 多包安装，也有 Makefile/Grunt 产物打包流程。

### 5.1 开发态构建（依赖准备）

- `npm run build`：安装各子包依赖（等价“准备开发环境”）

### 5.2 发行态打包

- `make all`：
  - 调用 Grunt 产出 server build
  - 拷贝 schema、license、branding、tools 等到目标目录
  - 写入 build version/build number/build date
- `make install`：安装到文档服务目录并准备运行所需目录/权限

> 也就是说：**npm scripts 偏开发依赖与测试，Makefile 偏发行打包与安装**。

---

## 6) 对外提供的服务能力列表（HTTP/协同）

### 6.1 健康与基础信息

- `GET /index.html`：服务状态与版本信息
- `GET /healthcheck`：健康检查

### 6.2 协同与命令

- `GET/POST /coauthoring/CommandService.ashx`
- `POST /command`

### 6.3 转换能力

- `GET/POST /ConvertService.ashx`
- `POST /converter`

### 6.4 上传/下载/保存/打印

- `POST /upload/:docid*`
- `POST /downloadas/:docid`
- `POST /savefile/:docid`
- `GET /printfile/:docid/:filename`
- `GET/POST /downloadfile/:docid`

### 6.5 WOPI 与扩展

- 主服务中接入 `wopiClient` 能力与相关配置项
- 具备 AI 代理处理模块 `ai/aiProxyHandler`

---

## 7) 你问的“主服务和子服务在 Node 里怎么搞”的答案

一句话：**进程拆分 + 队列解耦 + cluster 扩展 + Common 共享基础设施**。

### 7.1 进程模型

- **主服务 DocService**：单独 Node 进程，直接对外提供 HTTP/Socket.IO
- **子服务 FileConverter**：独立 Node 进程（master）+ 多 worker 子进程
- **子服务 SpellChecker**：独立 Node 进程（master）+ worker 子进程

### 7.2 服务间协作

- DocService 把转换需求转为任务（通过队列）
- FileConverter 消费任务并处理，再回写结果
- Common 抽象存储、队列、租户、配置，使各服务代码共享同一套基础能力

### 7.3 为什么这样设计

- 协同链路与转换链路解耦，避免互相阻塞
- 转换服务可按 CPU/license 横向扩 worker
- 故障隔离更清晰（转换挂了不等于主协同服务挂）
- 运维上可单独扩容热点子服务

---

## 8) 快速结论

这是一个典型“**主服务（协同网关）+ 多子服务（转换/拼写）+ 基础设施共享层**”的 Node.js 后端系统。开发时以 DocService 为观察中心，问题定位按链路分到 FileConverter 或 SpellChecker，再结合 Common 配置与队列/存储排查。


---

## 9) 有没有调用外部服务？在哪里、如何调用？

有，而且不少。可以分成“基础设施类外部服务”和“业务集成类外部服务”。

### 9.1 基础设施类

1) **数据库（PostgreSQL/MySQL/MSSQL/Oracle/Dameng）**
- 在 `DocService/sources/databaseConnectors/*` 中连接。
- 例如 PostgreSQL 通过 `pg.Pool(connectionConfig)` 建连接池，连接信息来自 `services.CoAuthoring.sql` 配置（host/port/user/password/dbName）。
- 调用方式：在各 connector 的 `sqlQuery` 中执行 SQL。

2) **消息队列（RabbitMQ / ActiveMQ）**
- 在 `Common/sources/taskqueueRabbitMQ.js` 中统一接入。
- 调用方式：
  - RabbitMQ：`rabbitMQCore.connetPromise` + `assertQueue` + `consume` + `ack`
  - ActiveMQ：`activeMQCore.connetPromise` + `openSender/openReceiver`
- 作用：DocService 与 FileConverter 之间的异步任务流转。

3) **对象存储（S3 / Azure Blob）**
- S3 在 `Common/sources/storage/storage-s3.js`，Azure 在 `Common/sources/storage/storage-az.js`。
- 调用方式：
  - S3：AWS SDK 的 `S3Client` + `GetObject/PutObject/CopyObject/...`
  - Azure：`BlobServiceClient` + `download/upload/syncCopyFromURL/...`

4) **SMTP 邮件服务**
- 在 `Common/sources/mailService.js`。
- 调用方式：`nodemailer.createTransport(...)` 创建 transporter，`sendMail(...)` 发信。

5) **Redis**
- 配置在 `services.CoAuthoring.redis`，用于编辑状态/统计等（由 editorData/editorStat 等模块使用）。

### 9.2 业务集成类

1) **外部 HTTP/HTTPS 服务（通用调用）**
- 在 `Common/sources/utils.js` 中通过 `axios` 统一发起 GET/POST/任意方法请求。
- 调用方式：`downloadUrlPromise` / `postRequestPromise` / `httpRequest`。
- 安全控制：
  - 受 `externalRequest.action`、`externalRequest.directIfIn` 配置控制。
  - 可开启私网地址拦截（`request-filtering-agent`），也可配置代理 `proxyUrl`/`proxyUser`/`proxyHeaders`。

2) **WOPI Host 生态**
- 在 `DocService/sources/server.js` 暴露 WOPI 相关路由，具体逻辑在 `DocService/sources/wopiClient.js`。
- 通过 `utils` 的 HTTP 能力与外部 WOPI 主机做文件信息、内容读写、锁等交互（并有 `wopi.*` 配置控制）。

3) **AI 代理（可选）**
- 在 `DocService/sources/server.js` 通过 `/ai-proxy` 路由接入 `aiProxyHandler.proxyRequest`。
- 是否真正调用外部 AI Provider，取决于 `aiSettings` 与代理配置。

4) **PDF 云签名服务（可选）**
- 在 `FileConverter/sources/converter.js` 中，若开启配置会调用：
  - AWS KMS（`signPdfFileKms`）
  - CSC 服务（`signPdfFileCsc`）
- 触发条件由 `FileConverter.converter.signing.awsKms` 或 `...signing.csc` 配置决定。

### 9.3 结论（直接回答你的问题）

- **有调用外部服务。**
- **在哪里调用：**主要集中在 `Common/sources/utils.js`（通用 HTTP）、`Common/sources/storage/*`（云存储）、`Common/sources/taskqueueRabbitMQ.js`（MQ）、`Common/sources/mailService.js`（SMTP）、`DocService/sources/wopiClient.js`（WOPI 集成）、`FileConverter/sources/converter.js`（KMS/CSC）。
- **如何调用：**通过统一配置 + SDK/HTTP 客户端（axios、AWS SDK、Azure SDK、nodemailer、AMQP 客户端）实现，并受安全配置（私网拦截、代理、token/header）约束。


---

## 10) 是否通过 License 做并发限制？有，且有两层

你问的这个点是对的：代码里确实有基于 license 的并发/容量限制逻辑，主要在 **DocService 鉴权层** 和 **FileConverter worker 数量层**。

### 10.1 DocService：编辑/查看并发与用户数限制（强相关）

位置：`DocService/sources/DocsCoServer.js`

- 在用户连接鉴权流程中调用 `_checkLicenseAuth(...)`，用于判断当前连接是否还能进入编辑/LiveViewer。  
- `_checkLicenseAuth` 内部按 license 规则做两类限制：
  1. **按用户数限制**（`usersCount` / `usersViewCount`）
  2. **按连接数限制**（`connections` / `connectionsView`）
- 达到上限时，会把 license 结果改成对应限制码（如 `UsersCount` / `Connections` 等），后续会将连接降级或拒绝编辑能力。
- 还会根据 `license.warning_limit_percents` 做阈值告警通知（接近上限时报警）。

可定位代码片段：
- 鉴权调用：`_checkLicenseAuth(...)` 在 auth 过程中执行
- 限制实现：`_checkLicenseAuth` 中对 users/connections 的比较与分支

### 10.2 FileConverter：按 license 限制 worker 并发（进程并行度）

位置：`FileConverter/sources/convertermaster.js`

- master 进程读取 license 后，worker 数按 `min(licenseInfo.count, CPU*maxprocesscount)` 计算。
- 这意味着即使机器 CPU 很多，也会被 license 里的 count 上限卡住。
- license 文件变化会触发 `updateLicense()`，随后动态增减 worker。

### 10.3 相关配置入口（辅助定位）

- `Common/config/default.json` 中可以看到许可证告警阈值配置：`license.warning_limit_percents`
- `DocsCoServer.js` 会读取这个阈值并触发 `notification` 模板告警。

### 10.4 一句话总结

- **有并发限制**，而且是“编辑/查看会话并发限制 + 转换 worker 并发限制”双路径。  
- 如果你要排查“为什么超并发后有人变只读/连不上、或者转换吞吐上不去”，优先看这两个位置。
