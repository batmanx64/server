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
