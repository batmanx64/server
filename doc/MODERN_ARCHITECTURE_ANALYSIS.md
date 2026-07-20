# 现代化架构分析 — ONLYOFFICE DocumentServer 重构方案

> 目标：面向自学 & 企业级，用最新技术栈重构

---

## 一、后端架构选型：分体式 vs 单体 vs 模块化单体

### 1.1 三种架构对比

```
┌────────────────────────────────────────────────────────────────────┐
│ 方案 A: 全分体微服务 (当前架构)                                      │
│                                                                     │
│  DocService ──HTTP──► FileConverter (独立进程)                       │
│       │              SpellChecker (独立进程)                         │
│       │              Metrics     (独立进程)                         │
│       └──RabbitMQ──► 消息队列通信                                    │
│                                                                     │
│  ✅ 独立扩缩容 ✅ 故障隔离 ✅ 部署灵活                                │
│  ❌ 调试困难  ❌ 运维复杂  ❌ 开发效率低                              │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│ 方案 B: 全单体 (Spring Boot 风格)                                   │
│                                                                     │
│  单一进程:                                                          │
│  ┌─────────────────────────────────────────────┐                   │
│  │  app.js (Express)                           │                   │
│  │  ├── DocService    (路由+控制器)              │                   │
│  │  ├── FileConverter (worker_thread)           │                   │
│  │  └── SpellChecker  (worker_thread)           │                   │
│  └─────────────────────────────────────────────┘                   │
│                                                                     │
│  ❌ 无法独立扩缩容 ❌ 一个模块 OOM 全挂                             │
│  ✅ 开发调试简单 ✅ 部署简单                                         │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│ 方案 C: 模块化单体 + 关键路径剥离 (推荐 🏆)                         │
│                                                                     │
│  pnpm monorepo                                                      │
│  ┌─────────────────────────────────────────────┐                   │
│  │ packages/                                   │                   │
│  │  ├── core/           (类型定义、工具函数)     │                   │
│  │  ├── storage/        (存储抽象 FS/S3/Azure)  │                   │
│  │  ├── messaging/      (消息队列抽象)          │                   │
│  │  ├── database/       (数据库连接器 + ORM)     │                   │
│  │  ├── collaboration/  (协同引擎 — Socket.IO)   │                   │
│  │  ├── conversion/     (格式转换编排)           │                   │
│  │  ├── spellcheck/     (拼写检查)               │                   │
│  │  ├── wopi/           (WOPI 协议集成)          │                   │
│  │  └── api-gateway/    (统一 HTTP 入口)         │                   │
│  └─────────────────────────────────────────────┘                   │
│                                                                     │
│  部署时按需组合:                                                     │
│  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐                              │
│  │ API  │ │ Conv │ │Spell │ │Metrics│                              │
│  │Gate  │ │Worker│ │Worker│ │       │                              │
│  └──────┘ └──────┘ └──────┘ └──────┘                              │
│                                                                     │
│  ✅ 开发期单体效率 ✅ 部署期微服务灵活                              │
│  ✅ 共享类型定义 ✅ 单步调试跨包调用                                │
└────────────────────────────────────────────────────────────────────┘
```

### 1.2 推荐方案：pnpm monorepo + 模块化单体

**开发期**：所有代码在一个仓库，TypeScript 包间直接引用，`tsc --watch` 秒级热重载

**部署期**：通过 Dockerfile 选择打包哪些包：

```dockerfile
# 打包全功能服务
FROM node:22 AS api-gateway
COPY packages/core packages/storage packages/messaging packages/database packages/collaboration packages/conversion packages/spellcheck packages/wopi packages/api-gateway
RUN pnpm build
CMD ["node", "packages/api-gateway/dist/index.js"]

# 只打包转换 Worker
FROM node:22 AS converter-worker
COPY packages/core packages/storage packages/messaging packages/conversion
RUN pnpm build
CMD ["node", "packages/conversion/dist/worker.js"]
```

**决策矩阵**：

| 标准 | 全微服务 | 全单体 | 模块化单体 (推荐) |
|------|----------|--------|-------------------|
| 学习曲线 | 陡峭 | 简单 | **中等** |
| 开发效率 | 低（跨服务调试） | 高 | **高** |
| 独立扩缩 | ✅ | ❌ | **✅ (打包时选择)** |
| 故障隔离 | ✅ | ❌ | **✅ (进程级隔离)** |
| 代码复用 | 难 | 容易 | **最容易 (同仓库)** |
| 企业级成熟度 | 高 | 低 | **高** |

---

## 二、web-apps 改造方案：Backbone → React/Vue

### 2.1 现状分析

```
当前 web-apps 架构:

web-apps/
├── apps/
│   ├── api/documents/api.js     ← 公共 API 入口 (DocsAPI.DocEditor)
│   ├── common/                  ← Backbone MVC 框架 (~5 万行)
│   │   ├── Gateway.js           ← postMessage 通信桥
│   │   └── main/lib/            ← 核心组件库
│   ├── documenteditor/          ← 文字编辑器
│   ├── spreadsheeteditor/       ← 表格编辑器
│   ├── presentationeditor/      ← 演示编辑器
│   ├── pdfeditor/               ← PDF 编辑器
│   └── visioeditor/             ← 流程图编辑器
└── vendor/                      ← jQuery, Backbone, Underscore, ACE, Monaco

技术栈: Backbone.js + jQuery + RequireJS (AMD) + Less
代码量: ~20 万行
```

### 2.2 改造策略

**不适合"一刀切"重写** — 20 万行 DOM 操作代码、5 个编辑器、深度耦合 sdkjs。推荐 **渐进式替换**。

```
阶段一: 基础设施替换 (3-4 个月)
┌────────────────────────────────────────────┐
│ 保留 Backbone 业务逻辑不变                    │
│ 底层替换:                                    │
│  ├── RequireJS ──► Vite (ESM 模块加载)       │
│  ├── jQuery    ──► 原生 DOM / Preact        │
│  ├── Less      ──► Tailwind CSS / CSS Modules│
│  └── Grunt     ──► Vite + pnpm              │
└────────────────────────────────────────────┘

阶段二: 核心组件替换 (3-6 个月)
┌────────────────────────────────────────────┐
│ 公共组件逐个替换:                             │
│  ├── Gateway.js ──► 保留 (postMessage 稳定)  │
│  ├── 工具栏 ──► React/Vue 组件               │
│  ├── 对话框 ──► React/Vue 组件               │
│  ├── 菜单栏 ──► React/Vue 组件               │
│  └── 状态栏 ──► React/Vue 组件               │
│ Backbone view 逐步包裹 React 组件             │
└────────────────────────────────────────────┘

阶段三: 编辑器独立化 (6-12 个月)
┌────────────────────────────────────────────┐
│ 每个编辑器发布独立 npm 包:                    │
│  ├── @onlyoffice/editor-document           │
│  ├── @onlyoffice/editor-spreadsheet        │
│  ├── @onlyoffice/editor-presentation       │
│  └── @onlyoffice/editor-pdf                │
│ 均共享: @onlyoffice/editor-core            │
│ 均使用: @onlyoffice/sdkjs (下一节)          │
└────────────────────────────────────────────┘
```

### 2.3 React vs Vue 选择

| 维度 | React | Vue |
|------|-------|-----|
| 企业生态 | **最大** (Meta) | 中等 |
| 编辑器类项目案例 | **Rich Text 框架丰富** (Slate, Prosemirror) | 较少 |
| TypeScript | **原生支持** | 需要 `defineComponent` 包裹 |
| 渐进式迁移 | **🍕 Micro-Frontend 方案成熟** (Module Federation) | 也可以但生态略弱 |
| 学习资源 | 丰富 | 丰富 |

**推荐：React**，理由：
1. `Gateway.js` 作为 postMessage 桥，天然适配微前端 — React 的 **Module Federation** 方案最成熟
2. 编辑器核心 (sdkjs) 是 Canvas 渲染，不与 React VDOM 冲突，可以 `<canvas ref={...}>` 嵌入
3. 未来可拆分为独立 npm 包，React 的组件生态更丰富

### 2.4 关键技术决策

```typescript
// 1. DocsAPI 保持为纯函数接口 (不依赖框架)
//    api/documents/api.ts
export interface DocsAPIConfig {
  document: DocumentConfig
  editorConfig: EditorConfig
  events: EditorEvents
}

export function DocEditor(placeholder: string | HTMLElement, config: DocsAPIConfig): Editor {
  // 返回 Editor 实例，不绑定 React/Vue
}

// 2. React 组件包装
//    components/Editor.tsx
function Editor({ config }: { config: DocsAPIConfig }) {
  const containerRef = useRef<HTMLDivElement>(null)

  useEffect(() => {
    const editor = DocEditor(containerRef.current!, config)
    return () => editor.destroy()
  }, [config])

  return (
    <div className="editor-wrapper">
      <Toolbar />
      <div ref={containerRef} className="editor-canvas" />
      <StatusBar />
    </div>
  )
}

// 3. 微前端集成 (可选)
//    如果主应用是 Vue/Angular，通过 Gateway postMessage 通信
//    无需改变 DocsAPI 接口
```

---

## 三、sdkjs 适配新架构

### 3.1 现状分析

```
sdkjs/ 的核心:
┌────────────────────────────────────┐
│  1. Canvas 渲染引擎 (自定义)         │ ← 核心价值，保留
│  2. OOXML 解析器 (自定义)           │ ← 核心价值，保留
│  3. OT 协同引擎 (Collaborative...)  │ ← 核心价值，保留
│  4. 字体引擎 (libfont/, WASM)      │ ← 核心价值，保留
│  5. API 层 (api.js 16K 行)         │ ← 需要拆分
│  6. 构建系统 (Grunt + Closure)     │ ← 需要替换
│  7. 模块系统 (文件拼接)             │ ← 需要替换
└────────────────────────────────────┘
```

### 3.2 改造策略

```
当前: 文件拼接 + 全局命名空间 (Asc, AscCommon)
sdk-all.js = common/*.js + word/*.js + cell/*.js + slide/*.js (全部在一个文件)

目标: ESM 模块化, 按需加载
```

**分三步走：**

#### 3.2.1 阶段一：构建系统替换 (1-2 个月)

```
Grunt + Google Closure Compiler ──► Vite + esbuild + TypeScript

步骤:
1. 保留所有 .js 文件不变
2. 用 Vite 的 rollup-plugin-glob-import 自动收集当前 configs/*.json 的
   文件列表到入口
3. 输出保持不变 (sdk-all.js)，但构建工具换成 Vite

收益: 构建时间从 ~60s 降到 ~3s，HMR 可用
```

#### 3.2.2 阶段二：ESM 模块化 (3-6 个月)

```typescript
// sdkjs/src/word/editor.ts
// 从全局命名空间改为 ESM 导出

// 当前 (全局污染):
// AscCommon.CollaborativeEditingBase

// 目标 (ESM):
export { CollaborativeEngine } from '../collaboration/engine'
export { DocumentModel } from './model/document'
export { WordRenderer } from './renderer/canvas'

// 统一的 SDK 入口:
// sdkjs/src/index.ts
export { createEditor } from './editor-factory'
export type { EditorConfig, DocumentType } from './types'
```

**关键：Canvas 渲染引擎保持独立** — 它不依赖任何 UI 框架，通过 `<canvas>` 元素嵌入 React/Vue 组件。

#### 3.2.3 阶段三：按需加载 + Tree-shaking (3-6 个月)

```typescript
// 用户安装时只需导入需要的编辑器类型
import { createDocumentEditor } from '@onlyoffice/sdkjs/word'
import { createSpreadsheetEditor } from '@onlyoffice/sdkjs/cell'

// 而不是现在的:
// <script src="sdk-all.js"></script> (1.5MB+)
```

### 3.3 最终的包结构

```
@onlyoffice/sdkjs/
├── core/                    ← 所有编辑器共享
│   ├── canvas-renderer/     ← Canvas 渲染引擎
│   ├── ooxml-parser/        ← OOXML 解析
│   ├── collaboration/       ← OT 协同引擎
│   ├── font-engine/         ← 字体引擎 (WASM)
│   └── types/               ← 共享类型
├── word/                    ← 文字编辑器
│   ├── model/
│   ├── renderer/
│   └── api.ts
├── cell/                    ← 表格编辑器
├── slide/                   ← 演示编辑器
├── pdf/                     ← PDF 编辑器
├── visio/                   ← 流程图编辑器
└── plugin-system/           ← 插件系统
```

---

## 四、整体架构蓝图

```
┌─────────────────────────────────────────────────────────────────────┐
│                      pnpm monorepo                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  packages/                                                          │
│  ├── @onlyoffice/core           类型定义、工具函数                    │
│  ├── @onlyoffice/storage        存储抽象 (FS/S3/Azure/R2)            │
│  ├── @onlyoffice/database       Drizzle ORM + 连接管理               │
│  ├── @onlyoffice/messaging      消息队列 (Redis Streams)             │
│  ├── @onlyoffice/collaboration  Socket.IO 协同引擎                   │
│  ├── @onlyoffice/conversion     x2t 转换编排                        │
│  ├── @onlyoffice/spellcheck     拼写检查                             │
│  ├── @onlyoffice/wopi           WOPI 协议实现                        │
│  ├── @onlyoffice/api-gateway    统一入口 (Hono/Fastify)              │
│  ├── @onlyoffice/sdkjs          Canvas 编辑引擎 (ESM)                │
│  └── @onlyoffice/web-apps       编辑器 UI (React)                    │
│                                                                     │
│  部署组合:                                                            │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐                │
│  │  Full Server │ │  Conv Worker │ │  Frontend    │                │
│  │  (API + Col- │ │  (conversion │ │  (Vite SPA)  │                │
│  │   laboration)│ │   + spell)   │ │              │                │
│  │  Hono/Socket │ │  Node worker │ │  React + SDK │                │
│  │  + DB + Stor │ │  thread     │ │              │                │
│  └──────────────┘ └──────────────┘ └──────────────┘                │
│         │                │                  │                       │
│         └────────────────┼──────────────────┘                       │
│                          ▼                                          │
│                 ┌──────────────────┐                                │
│                 │  x2t (C++ 原生)  │  ← 唯一无法替换的组件            │
│                 └──────────────────┘                                │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 五、技术栈速查表

| 层次 | 技术选型 | 原因 |
|------|----------|------|
| 运行时 | **Node.js 22 LTS** | Socket.IO 不可替代 |
| 语言 | **TypeScript 5.x strict** | 类型安全 |
| Monorepo | **pnpm workspace** | 最快、严格依赖隔离 |
| Web 框架 | **Hono** | 比 Fastify 更轻、Edge 兼容 |
| WebSocket | **Socket.IO 4.x** | 协同编辑的核心依赖 |
| ORM | **Drizzle** | 类型安全、SQL-like、零运行时 |
| 消息队列 | **Redis Streams** | 比 RabbitMQ 轻、自带持久化 |
| 缓存 | **ioredis + RedisOM** | 已有 Redis 依赖 |
| 存储 | **S3 SDK + tus** | 标准化、断点续传 |
| 前端框架 | **React 19 + Vite** | 企业生态、微前端成熟 |
| 前端组件 | **Radix UI / shadcn/ui** | 无样式、无障碍 |
| 测试 | **Vitest** | 快、ESM 原生 |
| CI/CD | **GitHub Actions** | 生态最好 |
| 容器化 | **Docker Compose + Dockerfile** | 标准 |

---

## 六、自学路线图

```
第 1 步 (1-2 月): TypeScript + Hono + Drizzle
  目标: 用 Hono + Drizzle 重写 server/Common 中的配置和数据库模块
  产出: @onlyoffice/database, @onlyoffice/core

第 2 步 (2-3 月): Socket.IO + Redis Streams
  目标: 替换 RabbitMQ → Redis Streams，理解消息队列设计
  产出: @onlyoffice/messaging

第 3 步 (3-4 月): 前端替换 RequireJS → Vite
  目标: 让 web-apps 跑在 Vite 上，保留 Backbone 逻辑
  产出: 可 HMR 的开发环境

第 4 步 (4-6 月): React 组件替换
  目标: 替换工具栏、对话框等独立 UI 组件
  产出: React 版本的公共组件库

第 5 步 (6-9 月): sdkjs ESM 化
  目标: sdkjs 输出 ESM 包，可按需加载
  产出: @onlyoffice/sdkjs

第 6 步 (9-12 月): 编辑器独立 npm 包
  目标: 每个编辑器可单独安装
  产出: @onlyoffice/editor-document 等
```

---

## 七、关键风险与应对

| 风险 | 程度 | 应对 |
|------|------|------|
| sdkjs 20 万行 JS 无类型 | 🔴 | 渐进加 JSDoc → `// @ts-check`，不改逻辑 |
| web-apps 5 个编辑器同步改造 | 🔴 | 选 documenteditor 做 pilot，其余沿用 |
| x2t C++ 二进制绑定 | 🟡 | 保持 child_process 调用，不改变接口 |
| Gateway postMessage 协议 | 🟡 | 保留，这是 SDK 的公共 API 契约 |
| 协作编辑的 OT 算法 | 🟢 | sdkjs 已有成熟实现，不动 |
| Canvas 渲染引擎 | 🟢 | sdkjs 核心资产，只做 ESM 包装不改逻辑 |

---

**总结：用模块化单体降低开发复杂度，用 pnpm monorepo 统一管理，后端保留 Socket.IO，前端渐进迁移 React，sdkjs 只做构建和模块化改造不动内核。**
