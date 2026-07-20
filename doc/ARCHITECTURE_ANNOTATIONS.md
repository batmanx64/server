# 架构注释索引

代码中各文件已标注 `// === [ARCHITECTURE]` 注释，本文档是索引速查表。

## 总体架构

```
Browser ──HTTP──┐
                ▼
         ┌───────────┐  Socket.IO  ┌─────────────┐
         │ DocService ├────────────► 浏览器客户端  │
         │(Express +  │             └─────────────┘
         │ Socket.IO) │──┐
         └─────┬─────┘  │ RabbitMQ
               │        │ 任务队列
         HTTP  │        ▼
         ┌─────▼─────┐ ┌──────────┐
         │ FileConv. │ │SpellCheck│
         │ (Cluster) │ │(Cluster) │
         └───────────┘ └──────────┘
               │
         ┌─────▼─────┐
         │  Metrics   │ (StatsD)
         └───────────┘

DocService ──► PostgreSQL/MySQL/MSSQL/Oracle/Dameng
DocService ──► Redis (presence/force-save)
DocService ──► FS/S3/Azure (文件存储)
DocService ◄──► RabbitMQ fanout (跨实例广播)
```

## 设计模式 → 源文件

| 模式 | 文件 | 说明 |
|------|------|------|
| **微服务 (进程分离)** | 所有入口文件 | 4 个独立 Node 进程: DocService, FileConverter, SpellChecker, Metrics |
| **Context 对象模式** | `Common/sources/operationContext.js` | 每次请求创建 Context，携带 tenant/docId/userId/shardKey 贯穿调用链 |
| **Strategy — 存储** | `Common/sources/storage/storage-base.js` | 统一接口, 运行时按 `name` 分发到 fs/s3/az |
| **Strategy — 数据库** | `DocService/.../databaseConnectors/baseConnector.js` | Factory 加载 mysql/postgres/mssql/oracle/dameng |
| **Strategy — 队列** | `Common/sources/taskqueueRabbitMQ.js` | 按 `queue.type` 切换 RabbitMQ/ActiveMQ |
| **Strategy — 编辑数据** | `DocService/sources/DocsCoServer.js:109` | 按配置切换 内存/Redis |
| **Factory — 数据库** | `DocService/.../baseConnector.js:60-78` | switch 加载对应 connector |
| **任务队列** | `Common/sources/taskqueueRabbitMQ.js` | DocService 提交 → FileConverter 消费 |
| **Pub/Sub** | `DocService/sources/pubsubRabbitMQ.js` | RabbitMQ fanout，多实例广播协同事件 |
| **Cluster Master-Worker** | `FileConverter/sources/convertermaster.js` | Master fork Worker，按许可限制数量，崩溃重启 |
| **Cluster Master-Worker** | `SpellChecker/sources/server.js` | 同上，Master 定时健康检查 |
| **观察者 (配置热更新)** | `DocService/sources/server.js:497-499` | `fs.watch` runtime.json + `fs.watchFile` 许可证文件 |
| **观察者 (许可证)** | `DocService/sources/server.js:146-147` | 日检 + 文件变更监听 |
| **限流 (bottleneck)** | `DocService/.../baseConnector.js:56-57` | getChanges 按 (tenant,docId) 分组限流 |
| **临界区** | `DocService/.../baseConnector.js:58,371-391` | 同一文档 deleteChanges 串行化 |
| **多租户** | `Common/sources/tenantManager.js` | 子域名 → 独立目录，每租户独立 config/secret/license |
| **JWT 三密钥** | `DocService/sources/server.js:137-143` | inbox/outbox/browser 三把独立密钥 |
| **Signed URL** | `Common/sources/storage/storage-base.js:158-213` | MD5 + 过期时间，nginx 安全分发缓存文件 |

## 数据流路径

```
用户打开文档:
  Browser ──HTTP──► /hosting/wopi/:type/:mode ──► wopiClient.getEditorHtml
       ──Socket.IO──► io.use (JWT验证) ──► io.on('connection') ──► auth ──► join room

编辑变更:
  Browser ──Socket.IO 'changes'──► DocsCoServer ──► pubsub.publish (广播给其他实例)
      │
      ├──► editorData (内存/Redis 记录状态)
      └──► sqlBase.insertChangesPromise (持久化到数据库)

保存文档:
  Browser ──Socket.IO 'forceSave'──► DocsCoServer ──► canvasService.saveFile
       ──► storage.putObject (写入 FS/S3/Azure)
       ──► taskResult (更新数据库状态)
       ──► HTTP callback (通知集成方)

格式转换:
  POST /converter ──► converterService ──► queue.addTask (RabbitMQ)
       ──► FileConverter Worker 消费 ──► 下载源文件 ──► x2t/docbuilder ──► 上传结果
       ──► queue.addResponse (响应队列) ──► HTTP callback

关闭文档:
  Browser 断开 Socket.IO ──► 等待 cfgAscSaveTimeOutDelay
       ──► 自动汇编 ──► canvasService.saveFile
       ──► 清理 editorData ──► 删除变更记录
```

## 关键配置节点

| 配置项 | 位置 | 用途 |
|--------|------|------|
| `services.CoAuthoring.token.enable.*` | `default.json` | JWT 三密钥开关 |
| `services.CoAuthoring.sql.type` | `default.json` | 数据库类型选择 |
| `storage.name` | `default.json` | 存储后端: storage-fs/s3/az |
| `queue.type` | `default.json` | 队列: rabbitmq / activemq |
| `services.CoAuthoring.server.editorDataStorage` | `default.json` | 编辑数据存储: 内存/Redis |
| `tenants.baseDomain` | `default.json` | 多租户域名基 |
| `FileConverter.converter.maxprocesscount` | `default.json` | Worker 数量系数 |
| `bottleneck.getChanges` | `default.json` | 数据库读取限流参数 |

## 自学指引

从最小的闭环 **SpellChecker** 开始:
1. `SpellChecker/sources/server.js` — Cluster + Express + SockJS
2. `SpellChecker/sources/spellCheck.js` — nodehun 拼写核心
3. 端到端: HTTP 请求 → 拼写检查 → 返回结果

再深入 **FileConverter** 的任务队列模式:
4. `Common/sources/taskqueueRabbitMQ.js` — 队列抽象
5. `Common/sources/rabbitMQCore.js` — 连接管理
6. `FileConverter/sources/converter.js` — 任务消费 + x2t 调用

最后理解 **DocService** 的完整协同流程:
7. `DocService/sources/server.js` — 入口路由
8. `DocService/sources/DocsCoServer.js` — Socket.IO 协同引擎
9. `DocService/sources/pubsubRabbitMQ.js` — 跨实例广播
10. `Common/sources/storage/storage-base.js` — 存储抽象
