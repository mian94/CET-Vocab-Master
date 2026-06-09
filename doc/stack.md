# 技术栈方案对比与选型

---

## 1. 技术需求分析

基于 PRD 和验收标准，提炼出以下关键技术需求：

| 需求来源 | 技术需求 | 技术要点 |
|---|---|---|
| NFR 3.2 | 移动端优先 | 响应式布局，触摸交互友好 |
| NFR 3.1 | Glassmorphism UI | 毛玻璃效果，需要 backdrop-filter 等 CSS 特性 |
| FR-3.3 | AI 流式生成 | 后端需支持 SSE/Stream，前端逐字渲染打字机效果 |
| NFR 3.2 | TTFB < 1.5s | AI 接口需流式转发，不能等全部生成完再返回 |
| FR-1.2 | 云端持久化 | MongoDB 数据库 |
| FR-1.1 | 用户认证 | 用户名/手机号/密码，JWT 或 Session |
| FR-4.1~4.4 | 管理端 | 独立 URL 路由，CRUD 操作 |
| 验收特性 5 | 管理端实时同步用户端 | 需要服务端推送机制（WebSocket / SSE / 轮询） |
| FR-2.3 | 音频播放 | Web Speech API 或第三方发音服务 |

---

## 2. 三种技术栈方案

### 方案 A：Nuxt 3（Vue 3 生态）

```
前端框架：Nuxt 3 + Vue 3 + TypeScript
CSS 框架：Tailwind CSS
后端方案：Nuxt Server Routes（内置 API 层）
数据库：MongoDB + Mongoose
认证：JWT（httpOnly Cookie）
AI 流式：Nuxt Server Routes 转发流式响应 + 前端 ReadableStream
实时同步：Server-Sent Events (SSE)
部署：Node.js 服务器 / Vercel / Docker
```

### 方案 B：Next.js（React 生态）

```
前端框架：Next.js 14 (App Router) + React 18 + TypeScript
CSS 框架：Tailwind CSS
后端方案：Next.js API Routes / Route Handlers
数据库：MongoDB + Mongoose（或 Prisma）
认证：NextAuth.js 或自建 JWT
AI 流式：Next.js Route Handlers + Vercel AI SDK（streamText）
实时同步：Server-Sent Events (SSE)
部署：Vercel（一键部署）/ Docker
```

### 方案 C：前后端分离（Vue 3 + Express）

```
前端：Vue 3 + Vite + Vue Router + Pinia + TypeScript
后端：Express.js / Fastify + TypeScript
CSS 框架：Tailwind CSS
数据库：MongoDB + Mongoose
认证：JWT（httpOnly Cookie）
AI 流式：Express 路由转发流式响应 + 前端 ReadableStream
实时同步：Socket.io（WebSocket）
部署：前端静态托管（Nginx/Vercel）+ 后端 Node.js 服务器
```

---

## 3. 方案对比

| 维度 | A. Nuxt 3 | B. Next.js | C. Vue + Express |
|---|---|---|---|
| **开发效率** | ⭐⭐⭐⭐⭐ 单仓全栈，开箱即用 | ⭐⭐⭐⭐ 单仓全栈，生态更大 | ⭐⭐⭐ 前后端两个项目，联调成本高 |
| **学习曲线** | ⭐⭐⭐⭐ Vue 语法简洁，中文资料丰富 | ⭐⭐⭐ React 概念更多，App Router 较新 | ⭐⭐⭐ 需分别掌握 Vue + Express |
| **AI 流式支持** | ⭐⭐⭐⭐ Server Routes 原生支持 | ⭐⭐⭐⭐⭐ Vercel AI SDK 一流支持 | ⭐⭐⭐⭐ 需手动实现流式转发 |
| **实时同步** | ⭐⭐⭐ SSE 简单实现 | ⭐⭐⭐ SSE 简单实现 | ⭐⭐⭐⭐ Socket.io 成熟方案 |
| **移动端适配** | ⭐⭐⭐⭐⭐ SSR 有利于首屏加载 | ⭐⭐⭐⭐⭐ SSR 有利于首屏加载 | ⭐⭐⭐ SPA 首屏较慢 |
| **部署复杂度** | ⭐⭐⭐⭐ Node 服务器或 Docker | ⭐⭐⭐⭐⭐ Vercel 一键部署 | ⭐⭐ 需分别部署前后端 |
| **中文社区** | ⭐⭐⭐⭐⭐ Vue 在中国用户基数大 | ⭐⭐⭐⭐ React 全球生态大 | ⭐⭐⭐⭐ Vue 社区活跃 |
| **项目规模适配** | ⭐⭐⭐⭐⭐ 中小型项目最佳 | ⭐⭐⭐⭐ 中大型项目更优 | ⭐⭐⭐ 适合需要独立扩展的场景 |
| **组件库可用性** | ⭐⭐⭐⭐ Nuxt UI / PrimeVue | ⭐⭐⭐⭐⭐ shadcn/ui / Ant Design | ⭐⭐⭐⭐ 同 Vue 生态 |

### 3.1 开发效率详解：单仓全栈的差异

方案 A 和方案 B 都是"单仓全栈"，但日常开发手感存在差异，主要体现在以下三个方面：

**① API Routes 写法复杂度**

Nuxt 3 Server Routes 一个文件即一个接口，写法极简：

```ts
// server/api/articles/index.get.ts
export default defineEventHandler(() => {
  return Article.find()
})
```

Next.js App Router Route Handlers 同样功能，需导出命名函数，写法更冗长：

```ts
// app/api/articles/route.ts
export async function GET(request: Request) {
  const articles = await Article.find()
  return Response.json(articles)
}
```

本项目有 15+ 个 API 接口，Nuxt 的写法每个文件少几行，累积差异明显。

**② 响应式数据处理的直觉成本**

本项目核心交互（文章阅读 + 高亮联动 + 气泡框弹出 + 打字机效果）全部依赖响应式数据。

Vue 3 的响应式是声明式的——改了 `ref` 的值，模板自动更新：

```ts
const text = ref('')
text.value += newChar  // 直接赋值，模板自动更新
```

React 的状态更新需要考虑闭包陷阱、依赖数组、effect 清理等问题：

```ts
const [text, setText] = useState('')
useEffect(() => {
  setText(prev => prev + newChar)  // 需注意闭包里的 stale state
}, [newChar])
```

对于打字机效果这种高频状态更新场景，Vue 的心智负担更低。

**③ 配置开箱即用程度**

Nuxt 3 零配置自动导入 `ref`、`computed`、`useRoute` 等常用函数，不需要手动 import。Next.js 每个文件都需要手动 import `useState`、`useMemo` 等。

**小结：** 不是"单仓全栈"这个特性有区别，而是框架本身的设计哲学导致日常开发的手感不同。Nuxt 3 在响应式交互密集 + 接口多 + 单人开发的场景下摩擦更少。唯一 Next.js 明显占优的是 AI 流式支持（Vercel AI SDK），但差距可以通过十几行代码弥补。

---

### 3.2 组件库可用性详解

方案 B 的 shadcn/ui / Ant Design 确实是目前设计审美和成熟度最好的组件库之一，但方案 A 并非没有对应选择：

**shadcn/ui 是 React 专属，不能直接用于 Vue。** 但有社区移植版 shadcn-vue（风格一致，成熟度稍低）。Ant Design 有独立的 Ant Design Vue 版本，可用于 Vue。

Vue 生态可用的组件库：

| 组件库 | 成熟度 | 特点 | 适合本项目 |
|---|---|---|---|
| **Nuxt UI** | ⭐⭐⭐⭐ | 专为 Nuxt 3 打造，Tailwind 深度集成，开箱即用 | ✅ 最契合 |
| **PrimeVue** | ⭐⭐⭐⭐⭐ | 90+ 组件，功能最全，类似 Ant Design 的定位 | ✅ 组件最丰富 |
| **shadcn-vue** | ⭐⭐⭐ | shadcn/ui 的 Vue 社区移植版 | ⚠️ 社区维护，成熟度稍低 |
| **Ant Design Vue** | ⭐⭐⭐⭐ | Ant Design 的 Vue 版，组件齐全 | ✅ 但设计风格偏"重" |
| **Naive UI** | ⭐⭐⭐⭐ | Vue 3 原生，中文社区活跃 | ✅ 但主题定制不如 Tailwind 灵活 |

**对本项目的实际影响：** PRD 定义的 UI 风格是极简纯白/浅灰 + 毛玻璃效果，需要的组件不多（侧边栏、气泡框、表单、按钮、流式打字机文本），用 Tailwind CSS 手写 + Nuxt UI 足够满足需求，未必需要引入重量级组件库。毛玻璃效果就是一行 CSS：`backdrop-filter: blur(12px);`

**结论：** shadcn/ui 的设计审美优势不构成本项目选 Next.js 的决定性理由。Nuxt UI 与 Nuxt 3 同团队维护，Tailwind 原生，更贴合极简毛玻璃的设计定位。

---

## 4. 推荐方案：方案 A — Nuxt 3

### 选型理由

**① 单仓全栈，开发效率最高**

Nuxt 3 的 Server Routes 让前后端在同一个项目中，无需跨项目联调。对于单人开发的中小型项目，这是最大的效率优势。

**② 移动端首屏体验好**

PRD 强调 Mobile-First，Nuxt 3 的 SSR 能力确保手机端首屏加载速度快，不依赖 JS 执行后才渲染内容。

**③ AI 流式生成天然支持**

Nuxt 3 的 Server Routes 基于 Nitro 引擎，原生支持流式响应（ReadableStream），转发 LLM 的 SSE 流只需十几行代码。前端配合 ReadableStream 逐字渲染即可实现打字机效果。

**④ 实时同步用 SSE 足够**

管理端实时同步用户端的需求，本质是"服务端单向推送"，SSE（Server-Sent Events）比 WebSocket 更轻量、实现更简单，且不需要双向通信。Nuxt Server Routes 原生支持 SSE。

**⑤ Vue 在中文生态优势明显**

Vue 3 + Nuxt 3 的中文文档完善，社区活跃，遇到问题更容易找到解决方案。

### 关键依赖版本

```
nuxt: ^3.x
vue: ^3.x
typescript: ^5.x
tailwindcss: ^3.x
mongoose: ^8.x
jsonwebtoken: ^9.x
bcryptjs: ^2.x
@vueuse/core: ^10.x（工具函数库）
```

### 推荐 LLM 接口

| 选项 | 优势 | 劣势 |
|---|---|---|
| **DeepSeek** | 中文能力强，价格低，兼容 OpenAI API 格式 | 偶有延迟波动 |
| 通义千问（Qwen） | 阿里云生态，国内访问快 | API 格式略有差异 |
| OpenAI (GPT-4o) | 质量最高，生态最成熟 | 国内需代理，成本较高 |

推荐首选 **DeepSeek**，理由：中文写作能力强、API 兼容 OpenAI 格式（迁移成本低）、价格友好。

---

## 5. 推荐架构

```
┌─────────────────────────────────────────────────┐
│                  Nuxt 3 应用                     │
│                                                   │
│  ┌──────────────┐    ┌──────────────────────────┐│
│  │   用户端页面   │    │       管理端页面          ││
│  │  / (首页)     │    │  /admin (独立路由)        ││
│  │  /read/:id    │    │  /admin/articles         ││
│  │  /generate    │    │  /admin/vocabulary       ││
│  └──────┬───────┘    │  /admin/templates        ││
│         │            └────────────┬─────────────┘│
│         │                         │               │
│  ┌──────▼─────────────────────────▼─────────────┐│
│  │              Nuxt Server Routes (API)         ││
│  │  /api/auth/*      认证接口                    ││
│  │  /api/articles/*  文章 CRUD                   ││
│  │  /api/vocab/*     词库 CRUD                   ││
│  │  /api/templates/* 模板 CRUD                   ││
│  │  /api/generate    AI 生成（流式）              ││
│  │  /api/sync        SSE 实时推送                ││
│  └──────────────────────┬───────────────────────┘│
│                         │                         │
└─────────────────────────┼─────────────────────────┘
                          │
           ┌──────────────┼──────────────┐
           │              │              │
     ┌─────▼─────┐  ┌────▼─────┐  ┌────▼─────┐
     │  MongoDB   │  │ DeepSeek │  │ Web Speech│
     │  数据库    │  │ LLM API  │  │  发音 API │
     └───────────┘  └──────────┘  └──────────┘
```

### 目录结构（建议）

```
CET_Vocab_Master/
├── nuxt.config.ts
├── package.json
├── tailwind.config.ts
├── .env
│
├── pages/                    # 用户端页面
│   ├── index.vue             # 首页/登录
│   ├── read/
│   │   └── [id].vue          # 阅读页
│   └── generate.vue          # AI 定制页
│
├── admin/                    # 管理端页面（Nuxt 自动注册为 /admin 路由）
│   ├── index.vue             # 管理端首页
│   ├── articles.vue          # 文章管理
│   ├── vocabulary.vue        # 词库管理
│   └── templates.vue         # AI 模板管理
│
├── components/               # 公共组件
│   ├── Sidebar.vue           # 侧边栏
│   ├── Popover.vue           # 查词气泡框
│   ├── ArticleReader.vue     # 文章阅读器
│   └── StreamingText.vue     # 流式打字机组件
│
├── server/                   # 后端 API
│   ├── api/
│   │   ├── auth/
│   │   │   ├── register.post.ts
│   │   │   ├── login.post.ts
│   │   │   └── me.get.ts
│   │   ├── articles/
│   │   │   ├── index.get.ts
│   │   │   ├── index.post.ts
│   │   │   ├── [id].put.ts
│   │   │   └── [id].delete.ts
│   │   ├── vocab/
│   │   │   ├── index.get.ts
│   │   │   ├── index.post.ts
│   │   │   ├── [id].put.ts
│   │   │   └── [id].delete.ts
│   │   ├── templates/
│   │   │   ├── index.get.ts
│   │   │   ├── index.post.ts
│   │   │   ├── [id].put.ts
│   │   │   └── [id].delete.ts
│   │   ├── generate.post.ts  # AI 流式生成
│   │   └── sync.get.ts       # SSE 实时推送
│   │
│   ├── middleware/
│   │   └── auth.ts           # JWT 鉴权中间件
│   │
│   └── utils/
│       ├── db.ts             # MongoDB 连接
│       └── jwt.ts            # JWT 工具函数
│
├── composables/              # 前端组合式函数
│   ├── useAuth.ts
│   ├── useArticles.ts
│   ├── useVocab.ts
│   └── useStreaming.ts
│
├── stores/                   # Pinia 状态管理
│   ├── auth.ts
│   └── reading.ts
│
└── doc/                      # 文档
    ├── prd.md
    └── Acceptance Criteria.md
```
