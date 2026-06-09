# 技术方案设计文档（开发文档）

> 本文档基于 PRD（`doc/prd.md`）、验收标准（`doc/Acceptance Criteria.md`）、技术选型（`doc/stack.md`）和架构决策（`doc/architecture-decisions.md`）生成，作为开发实施的唯一技术参考。

---

## 1. 技术栈总览

| 层级 | 技术 | 版本 | 用途 |
|---|---|---|---|
| 前端框架 | Nuxt 3 + Vue 3 | ^3.x | SSR、文件路由、全栈开发 |
| 类型系统 | TypeScript | ^5.x | 类型安全 |
| CSS 框架 | Tailwind CSS | ^3.x | 响应式布局、Glassmorphism |
| 状态管理 | Pinia | ^2.x | 全局状态（用户认证、阅读状态） |
| 工具函数 | @vueuse/core | ^10.x | 通用 Composition API 工具 |
| 数据库 | MongoDB | — | 云端持久化 |
| ODM | Mongoose | ^8.x | MongoDB 对象建模 |
| 认证 | JWT + bcryptjs | ^9.x / ^2.x | 用户认证与密码加密 |
| AI 接口 | DeepSeek API | — | LLM 文本生成（兼容 OpenAI 格式） |
| 发音 | Web Speech API | — | 浏览器内置语音合成，零成本 |
| Markdown 渲染 | markdown-it | ^14.x | 文章内容从 Markdown 解析为 HTML |

---

## 2. 架构设计

### 2.1 系统架构图

```
┌─────────────────────────────────────────────────────┐
│                    Nuxt 3 应用                       │
│                                                       │
│  ┌──────────────────┐    ┌──────────────────────────┐│
│  │     用户端页面     │    │       管理端页面          ││
│  │  / (登录/注册)    │    │  /admin (数据看板+入口)   ││
│  │  /read/:id        │    │  /admin/articles         ││
│  │  /generate        │    │  /admin/vocabulary       ││
│  └────────┬─────────┘    │  /admin/templates        ││
│           │               └────────────┬─────────────┘│
│           │                            │               │
│  ┌────────▼────────────────────────────▼─────────────┐│
│  │            Nuxt Server Routes (API)                ││
│  │                                                    ││
│  │  /api/auth/*          认证接口（管理员/用户共用）    ││
│  │  /api/articles/*      文章 CRUD                    ││
│  │  /api/vocab/*         词库 CRUD                    ││
│  │  /api/templates/*     模板 CRUD                    ││
│  │  /api/admin/stats     管理端统计数据                ││
│  │  /api/generate        AI 生成（SSE 流式输出）       ││
│  │                                                    ││
│  │  middleware/auth.ts    JWT 鉴权（所有接口共用）      ││
│  │  middleware/admin.ts   角色鉴权（管理端接口专用）    ││
│  └─────────────────────────┬─────────────────────────┘│
│                            │                           │
└────────────────────────────┼───────────────────────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
        ┌─────▼─────┐  ┌────▼─────┐  ┌────▼──────────┐
        │  MongoDB   │  │ DeepSeek │  │ Web Speech API │
        │  数据库    │  │ LLM API  │  │ (浏览器内置)   │
        └───────────┘  └──────────┘  └───────────────┘
```

### 2.2 数据流向

**用户阅读流程：**
```
用户访问 /read/:id
  → useFetch 拉取文章数据（/api/articles/:id）
  → useFetch 拉取当前级别词库（/api/vocab?level=cet4）
  → 前端匹配文章中的高亮词汇
  → 用户点击高亮词 → 气泡框展示释义 + 发音按钮
  → 用户切换级别 → useFetch 重新拉取对应级别词库 → 高亮重新匹配
```

**AI 生成流程：**
```
用户选择单词 + 输入主题 → POST /api/generate
  → Server Routes 组装 Prompt（含模板参考）
  → 转发 DeepSeek SSE 流式响应给前端
  → 前端 ReadableStream 逐字渲染打字机效果
  → 后端流式完成后自动保存文章到 Article 集合（type: custom, author: userId）
  → SSE 最后一条消息返回 articleId → 前端更新侧边栏"我的定制"目录
```

**从文章选词到 AI 生成流程：**
```
用户在 /read/:id 阅读页勾选文章中的单词
  → 点击"用这些词生成"按钮
  → readingStore.setPendingWords(selectedWords)
  → navigateTo('/generate')
  → 生成页 onMounted 检查 pendingWords → 自动填入 WordSelector → clearPendingWords()
  → 后续走正常 AI 生成流程
```

**数据刷新策略：**
- 用户端不使用 SSE，依赖 Nuxt 3 的 `useFetch` / `useAsyncData` 在路由切换时自动重新拉取
- 管理员更新内容后，用户在下次路由切换时即可看到最新数据

---

## 3. 数据模型设计

### 3.1 User（用户）

```ts
// types/user.ts
interface IUser {
  _id: string
  username: string          // 用户名，唯一
  phone: string             // 手机号，唯一
  password: string          // bcrypt 加密后的密码
  role: 'user' | 'admin'   // 角色，默认 'user'
  createdAt: Date
  updatedAt: Date
}
```

索引：`username`（唯一）、`phone`（唯一）

### 3.2 Article（文章）

```ts
// types/article.ts
interface IArticle {
  _id: string
  title: string                    // 文章标题
  content: string                  // 文章正文（Markdown 格式）
  level: 'cet4' | 'cet6'          // 所属等级
  topicTags: string[]              // 主题标签，如 ['#传统文化', '#科技发展']
  highlightedWords: string[]       // 需要高亮的词汇列表
  type: 'preset' | 'custom'       // preset=管理员预置, custom=用户AI生成
  author?: string                  // custom 类型时关联 userId
  order: number                    // 排序序号（用于侧边栏目录排序）
  createdAt: Date
  updatedAt: Date
}
```

索引：`level`、`type`、`author`、`{ level, order }`（复合索引）

### 3.3 Vocabulary（词汇）

```ts
// types/vocabulary.ts
interface IVocabulary {
  _id: string
  word: string               // 单词
  phonetic: string           // 音标，如 /səbˈstɪtjuːt/
  meaning: string            // 中文释义
  level: 'cet4' | 'cet6'    // 所属等级
  createdAt: Date
  updatedAt: Date
}
```

索引：`word`（唯一）、`level`、`{ word, level }`（复合索引）

### 3.4 AITemplate（AI 模板）

```ts
// types/template.ts
interface IAITemplate {
  _id: string
  title: string              // 模板标题
  content: string            // 模板正文
  topicTags: string[]        // 主题标签，用于匹配用户主题
  createdAt: Date
  updatedAt: Date
}
```

索引：`topicTags`

### 3.5 UserProgress（用户进度）

```ts
// types/user-progress.ts
interface IUserProgress {
  _id: string
  userId: string             // 关联用户
  articleId: string          // 关联文章
  mastered: boolean          // 是否已熟记
  createdAt: Date
  updatedAt: Date
}
```

索引：`{ userId, articleId }`（复合唯一索引）

### 3.6 AIGenerationLog（AI 生成日志）

```ts
// types/ai-generation-log.ts
interface IAIGenerationLog {
  _id: string
  userId: string             // 关联用户
  words: string[]            // 选用的单词列表
  topic: string              // 主题
  articleId?: string         // 生成的文章 ID
  createdAt: Date
}
```

索引：`userId`、`createdAt`（用于管理端统计）

---

## 4. API 接口设计

### 4.1 认证接口

所有接口前缀：`/api/auth`

| 方法 | 路径 | 说明 | 请求体 | 响应 |
|---|---|---|---|---|
| POST | `/api/auth/register` | 用户注册 | `{ username, phone, password }` | `{ token, user }` |
| POST | `/api/auth/login` | 登录（管理员/用户共用） | `{ username, password }` | `{ token, user: { ..., role } }` |
| GET | `/api/auth/me` | 获取当前用户信息 | — | `{ user }` |

**登录逻辑：**
- 接收用户名，查询数据库匹配 `username` 或 `phone`
- bcrypt 比对密码
- 成功后签发 JWT（payload 含 `userId`、`role`），写入 httpOnly Cookie
- 返回用户信息（含 `role`），前端根据 `role` 决定跳转 `/` 或 `/admin`

### 4.2 文章接口

所有接口前缀：`/api/articles`

| 方法 | 路径 | 权限 | 说明 | 查询参数/请求体 |
|---|---|---|---|---|
| GET | `/api/articles` | 用户 | 获取文章列表 | `?level=cet4&topic=传统文化` |
| GET | `/api/articles/:id` | 用户 | 获取单篇文章 | — |
| POST | `/api/articles` | 管理员 | 创建文章 | `{ title, content, level, topicTags, highlightedWords, order }` |
| PUT | `/api/articles/:id` | 管理员 | 更新文章 | `{ title?, content?, level?, topicTags?, highlightedWords?, order? }` |
| DELETE | `/api/articles/:id` | 管理员 | 删除文章 | — |

**查询逻辑：**
- 列表接口按 `level` 筛选，按 `order` 排序
- 用户端只返回 `type: 'preset'` 的文章；用户自己的 `type: 'custom'` 文章通过 `author` 字段单独查询（侧边栏"我的定制"目录）
- 管理端返回所有类型
- 用户不能直接调用 `POST /api/articles` 创建文章；AI 生成的文章由 `/api/generate` 接口在后端自动保存

### 4.3 词汇接口

所有接口前缀：`/api/vocab`

| 方法 | 路径 | 权限 | 说明 | 查询参数/请求体 |
|---|---|---|---|---|
| GET | `/api/vocab` | 用户 | 获取词汇列表 | `?level=cet4&search=sust&limit=50&offset=0` |
| POST | `/api/vocab` | 管理员 | 创建词条 | `{ word, phonetic, meaning, level }` |
| PUT | `/api/vocab/:id` | 管理员 | 更新词条 | `{ word?, phonetic?, meaning?, level? }` |
| DELETE | `/api/vocab/:id` | 管理员 | 删除词条 | — |

**查询逻辑：**
- 列表接口支持 `level` 筛选 + `search` 模糊搜索（按 `word` 字段）
- 支持分页（`limit` + `offset`），默认每页 50 条

### 4.4 AI 模板接口

所有接口前缀：`/api/templates`

| 方法 | 路径 | 权限 | 说明 | 查询参数/请求体 |
|---|---|---|---|---|
| GET | `/api/templates` | 管理员 | 获取模板列表 | `?topic=环境保护` |
| POST | `/api/templates` | 管理员 | 创建模板 | `{ title, content, topicTags }` |
| PUT | `/api/templates/:id` | 管理员 | 更新模板 | `{ title?, content?, topicTags? }` |
| DELETE | `/api/templates/:id` | 管理员 | 删除模板 | — |

### 4.5 AI 生成接口

| 方法 | 路径 | 权限 | 说明 |
|---|---|---|---|
| POST | `/api/generate` | 用户 | AI 流式生成短文（SSE） |

**请求体：**
```ts
{
  words: string[]    // 5-10 个单词
  topic: string      // 主题
  level: 'cet4' | 'cet6'
}
```

**响应：** SSE 流式输出，Content-Type 为 `text/event-stream`

**实现逻辑：**
1. 校验 `words.length` 在 5-10 范围内
2. 根据 `topic` 从 `AITemplate` 集合中匹配最相关的模板（按 `topicTags` 匹配）；若无匹配模板则跳过范文参考部分
3. 组装 Prompt（含模板风格参考 + 目标单词 + 主题 + 要求）
4. 调用 DeepSeek API（流式模式），将完整响应内容拼接为 `fullText`
5. 转发 SSE 流到前端的同时，将 `fullText` 逐步拼接
6. 流式完成后，后端自动将生成的文章保存到 `Article` 集合（`type: 'custom'`，`author` 为当前用户 ID，`highlightedWords` 为请求中的 `words`，`order` 为当前用户最大 order + 1）
7. 记录 `AIGenerationLog`（关联生成的文章 `articleId`）
8. 在 SSE 流的最后一条消息中返回 `data: {"articleId": "xxx"}`，前端据此更新侧边栏"我的定制"目录

**Prompt 模板结构：**
```
你是一位幽默风趣的英语写作老师。请参考以下范文的风格和结构，写一篇英语短文。

【范文参考】
${template.content}

【写作要求】
- 主题：${topic}
- 必须自然融入以下单词：${words.join(', ')}
- 字数：150-250 词
- 风格：幽默诙谐，适合大学生阅读
- 包含适合四六级写作/翻译的优秀句型
```

### 4.6 管理端统计接口

| 方法 | 路径 | 权限 | 说明 |
|---|---|---|---|
| GET | `/api/admin/stats` | 管理员 | 获取管理端统计数据 |

**响应：**
```ts
{
  totalArticles: number        // 总文章数
  articlesByLevel: { cet4: number, cet6: number }  // 按级别统计
  totalUsers: number           // 注册用户总数
  totalGenerations: number     // AI 生成总次数
}
```

### 4.7 用户进度接口

| 方法 | 路径 | 权限 | 说明 | 请求体 |
|---|---|---|---|---|
| GET | `/api/progress` | 用户 | 获取当前用户的所有进度 | — |
| PUT | `/api/progress/:articleId` | 用户 | 更新文章熟记状态 | `{ mastered: boolean }` |

---

## 5. 页面路由设计

### 5.1 用户端页面

| 路径 | 文件 | 布局 | 说明 | 鉴权 |
|---|---|---|---|---|
| `/` | `pages/index.vue` | `default` | 登录/注册同页（Tab 切换） | 未登录可访问 |
| `/read/:id` | `pages/read/[id].vue` | `default` | 文章阅读页 | 需登录 |
| `/generate` | `pages/generate.vue` | `default` | AI 短文定制页 | 需登录 |

**路由守卫逻辑：**
- 未登录用户访问 `/read/:id` 或 `/generate` → 跳转 `/`
- 已登录用户访问 `/` → 跳转到上次阅读的文章或第一篇文章

### 5.2 管理端页面

| 路径 | 文件 | 布局 | 说明 | 鉴权 |
|---|---|---|---|---|
| `/admin` | `admin/index.vue` | `admin` | 数据看板 + 快捷入口 | 需管理员 |
| `/admin/articles` | `admin/articles.vue` | `admin` | 文章管理 | 需管理员 |
| `/admin/vocabulary` | `admin/vocabulary.vue` | `admin` | 词库管理 | 需管理员 |
| `/admin/templates` | `admin/templates.vue` | `admin` | AI 模板管理 | 需管理员 |

**路由守卫逻辑：**
- 未登录用户访问 `/admin/*` → 跳转 `/`
- 已登录但 `role !== 'admin'` 的用户访问 `/admin/*` → 跳转 `/` 并提示无权限

---

## 6. 布局设计

### 6.1 用户端默认布局（`layouts/default.vue`）

```
┌─────────────────────────────────────┐
│  顶部导航栏                          │
│  [语境四六级]  [四级|六级]  [用户名]  │
├──────────┬──────────────────────────┤
│  侧边栏   │       主内容区            │
│          │                          │
│  #001    │    文章正文 / AI定制页     │
│  #002 ✓  │                          │
│  #003    │                          │
│  ...     │                          │
└──────────┴──────────────────────────┘
```

- 顶部导航栏：产品名称、四六级切换开关、用户信息/退出
- 侧边栏：文章目录列表，含编号、主题标签、已熟记状态图标
- 移动端：侧边栏默认隐藏，通过汉堡菜单按钮展开

### 6.2 管理端布局（`layouts/admin.vue`）

```
┌─────────────────────────────────────┐
│  管理端顶栏                          │
│  [管理后台]              [退出登录]  │
├──────────┬──────────────────────────┤
│  左侧菜单 │       主内容区            │
│          │                          │
│  📊 总览  │    当前页面内容            │
│  📝 文章  │                          │
│  📖 词库  │                          │
│  🤖 模板  │                          │
└──────────┴──────────────────────────┘
```

- 左侧菜单：总览、文章管理、词库管理、AI 模板管理
- 移动端：左侧菜单折叠为顶部汉堡菜单

---

## 7. 组件设计

### 7.1 公共组件

| 组件 | 文件 | 说明 |
|---|---|---|
| `Sidebar` | `components/Sidebar.vue` | 用户端侧边栏，展示文章目录列表 |
| `Popover` | `components/Popover.vue` | 查词气泡框，展示音标、释义、发音按钮 |
| `ArticleReader` | `components/ArticleReader.vue` | 文章阅读器，负责文本渲染和高亮匹配 |
| `StreamingText` | `components/StreamingText.vue` | 流式打字机组件，逐字渲染 AI 生成内容 |
| `WordSelector` | `components/WordSelector.vue` | 词汇选择器，支持从词库或文章中勾选单词 |
| `TopicInput` | `components/TopicInput.vue` | 主题输入框 + 快捷标签选择 |
| `LoginForm` | `components/LoginForm.vue` | 登录表单组件 |
| `RegisterForm` | `components/RegisterForm.vue` | 注册表单组件 |

### 7.2 管理端组件

| 组件 | 文件 | 说明 |
|---|---|---|
| `ArticleForm` | `components/admin/ArticleForm.vue` | 文章创建/编辑表单 |
| `VocabForm` | `components/admin/VocabForm.vue` | 词汇创建/编辑表单 |
| `TemplateForm` | `components/admin/TemplateForm.vue` | AI 模板创建/编辑表单 |
| `StatsCard` | `components/admin/StatsCard.vue` | 统计数据卡片 |

### 7.3 组件交互细节

**Popover 气泡框：**
- 点击高亮单词时，在单词上方弹出
- 内容：音标 + 中文释义 + 发音按钮
- 点击发音按钮 → 调用 `useAudio` composable → Web Speech API 播放
- 关闭方式：点击气泡框外部区域自动关闭
- 同一时间只展示一个气泡框

**StreamingText 打字机组件：**
- 接收 SSE 数据流，逐字符追加到显示区域
- 渲染时将目标单词用高亮标记包裹
- 生成过程中展示光标闪烁动画
- 生成完成或失败后隐藏光标

**管理端删除操作确认：**
- 管理员点击删除按钮时，使用 `window.confirm()` 弹出确认对话框
- 确认后执行删除 API 调用，取消则中止
- 后续迭代可替换为自定义 `ConfirmDialog.vue` 组件

**ArticleReader 文章阅读器：**
- 接收 Markdown 格式的文章内容和 `highlightedWords` 高亮词汇列表
- 渲染管线分两步（见第 10.4 节）：
  1. markdown-it 将 Markdown 解析为 HTML
  2. 遍历 HTML 纯文本节点，对匹配的单词包裹 `<mark>` 标签
- `<mark>` 标签绑定点击事件，触发 Popover
- 点击 Popover 外部区域时关闭当前 Popover
- 管理员写文章时使用标准 Markdown 语法，无需手动插入高亮标记

---

## 8. Composables 设计

| Composable | 文件 | 职责 |
|---|---|---|
| `useAuth` | `composables/useAuth.ts` | 用户认证状态管理：登录、注册、获取当前用户、登出 |
| `useArticles` | `composables/useArticles.ts` | 文章数据获取：列表、详情、按级别筛选 |
| `useVocab` | `composables/useVocab.ts` | 词汇数据获取：列表、搜索、按级别筛选 |
| `useStreaming` | `composables/useStreaming.ts` | AI 流式生成：发起 SSE 请求、逐字接收、错误处理、中断控制 |
| `useAudio` | `composables/useAudio.ts` | 发音播报：Web Speech API 封装、播放控制 |
| `useProgress` | `composables/useProgress.ts` | 进度管理：标记/取消熟记、获取进度列表 |

---

## 9. Store 设计（Pinia）

### 9.1 `stores/auth.ts`

```ts
interface AuthState {
  user: IUser | null
  token: string | null
  isLoggedIn: boolean
  isAdmin: boolean
}
```

操作：`login()`、`register()`、`logout()`、`fetchUser()`

### 9.2 `stores/reading.ts`

```ts
interface ReadingState {
  currentLevel: 'cet4' | 'cet6'
  articles: IArticle[]
  currentArticle: IArticle | null
  progress: Map<string, boolean>  // articleId → mastered
  pendingWords: string[]          // 从阅读页传递到生成页的待选单词
}
```

操作：`setLevel()`、`fetchArticles()`、`setCurrentArticle()`、`toggleMastered()`、`setPendingWords()`、`clearPendingWords()`

**跨页面传词流程（从文章选词 → AI 生成页）：**
1. 用户在阅读页（`/read/:id`）勾选文章中的单词
2. 点击"用这些词生成"按钮 → 调用 `readingStore.setPendingWords(selectedWords)`
3. 跳转到 `/generate` 页面
4. 生成页 `onMounted` 时检查 `readingStore.pendingWords`，如有值则自动填入 WordSelector 并调用 `clearPendingWords()`

---

## 10. 关键实现细节

### 10.1 JWT 认证流程

```
注册/登录成功
  → Server Routes 签发 JWT（payload: { userId, role }，有效期 7 天）
  → 写入 httpOnly Cookie（名称: token，HttpOnly: true，SameSite: Lax）
  → 前端通过 useAuth().fetchUser() 获取用户信息存入 Pinia

API 请求
  → Nuxt Server Middleware (auth.ts) 从 Cookie 读取 token
  → 验证 JWT 有效性，解析出 userId 和 role
  → 将用户信息挂载到 event.context.user
  → 管理端接口额外经过 admin.ts 中间件校验 role === 'admin'

登出
  → 清除 Cookie + 清除 Pinia 状态
```

### 10.2 AI 流式生成实现

**后端（`server/api/generate.post.ts`）：**

```ts
// 伪代码
export default defineEventHandler(async (event) => {
  const { words, topic, level } = await readBody(event)

  // 1. 校验单词数量
  if (words.length < 5 || words.length > 10) {
    throw createError({ statusCode: 400, message: '...' })
  }

  // 2. 匹配模板
  const template = await AITemplate.findOne({ topicTags: { $in: [topic] } })

  // 3. 组装 Prompt
  const prompt = buildPrompt(words, topic, template)

  // 4. 调用 DeepSeek API（流式模式）
  const response = await fetch('https://api.deepseek.com/v1/chat/completions', {
    method: 'POST',
    headers: { 'Authorization': `Bearer ${apiKey}`, 'Content-Type': 'application/json' },
    body: JSON.stringify({ model: 'deepseek-chat', messages: [...], stream: true })
  })

  // 5. 转发 SSE 流到前端
  setResponseHeaders(event, { 'Content-Type': 'text/event-stream', 'Cache-Control': 'no-cache' })
  return sendStream(event, response.body)
})
```

**前端（`composables/useStreaming.ts`）：**

```ts
// 伪代码
export function useStreaming() {
  const text = ref('')
  const isStreaming = ref(false)
  const error = ref<string | null>(null)

  async function generate(words: string[], topic: string, level: string) {
    isStreaming.value = true
    text.value = ''
    error.value = null

    try {
      const response = await fetch('/api/generate', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ words, topic, level })
      })

      const reader = response.body!.getReader()
      const decoder = new TextDecoder()

      while (true) {
        const { done, value } = await reader.read()
        if (done) break
        text.value += decoder.decode(value)
      }
    } catch (e) {
      error.value = '生成失败，请稍后重试'
    } finally {
      isStreaming.value = false
    }
  }

  function abort() { /* 中断流式读取 */ }

  return { text, isStreaming, error, generate, abort }
}
```

### 10.3 发音播报实现

```ts
// composables/useAudio.ts
export function useAudio() {
  function play(word: string) {
    const utterance = new SpeechSynthesisUtterance(word)
    utterance.lang = 'en-US'
    utterance.rate = 0.9
    speechSynthesis.speak(utterance)
  }

  return { play }
}
```

### 10.4 文章渲染管线与高亮匹配逻辑

文章内容为 Markdown 格式，渲染时需经过两步管线，确保 Markdown 语法和高亮逻辑互不干扰：

```
原始 Markdown 文本
  → 第一步：markdown-it 解析为 HTML（得到 <p>、<strong> 等标签）
  → 第二步：遍历 HTML 中的纯文本节点，对匹配的单词包裹 <mark> 标签
  → 最终输出到页面
```

**第一步：Markdown → HTML**

```ts
import MarkdownIt from 'markdown-it'
const md = new MarkdownIt()
const html = md.render(article.content)
```

**第二步：文本节点高亮替换**

只遍历文本节点（`nodeType === 3`），不触碰 HTML 标签节点，避免误匹配标签属性：

```ts
function highlightHtmlTextNodes(html: string, words: string[]): string {
  const container = document.createElement('div')
  container.innerHTML = html

  // 按单词长度降序排列，避免短词先匹配破坏长词
  const sorted = words.sort((a, b) => b.length - a.length)
  const regex = new RegExp(`\\b(${sorted.join('|')})\\b`, 'gi')

  const walker = document.createTreeWalker(container, NodeFilter.SHOW_TEXT)
  const textNodes: Text[] = []
  while (walker.nextNode()) textNodes.push(walker.currentNode as Text)

  for (const node of textNodes) {
    if (!regex.test(node.textContent!)) continue
    const span = document.createElement('span')
    span.innerHTML = node.textContent!.replace(
      regex,
      '<mark class="highlight" data-word="$1">$1</mark>'
    )
    node.parentNode!.replaceChild(span, node)
  }

  return container.innerHTML
}
```

**在 ArticleReader 组件中的调用：**

```ts
const renderedHtml = computed(() => {
  const html = md.render(props.article.content)
  return highlightHtmlTextNodes(html, props.article.highlightedWords)
})
```

**注意事项：**
- `<mark>` 标签通过事件委托绑定点击事件（绑定在文章容器上，通过 `data-word` 属性识别目标词），而非在渲染时逐个绑定
- SSR 兼容：`document.createTreeWalker` 仅在客户端可用，需在 `onMounted` 或 `process.client` 条件下调用

### 10.5 移动端适配策略

使用 Tailwind CSS 断点系统：

| 断点 | 宽度 | 布局调整 |
|---|---|---|
| 默认（手机） | < 768px | 侧边栏隐藏（汉堡菜单展开）、单列布局、气泡框全宽 |
| md | ≥ 768px | 侧边栏常驻展示、双栏布局 |
| lg | ≥ 1024px | 管理端左侧菜单常驻展示 |

---

## 11. 环境变量

```env
# .env
MONGODB_URI=mongodb://localhost:27017/cet-vocab
JWT_SECRET=your-jwt-secret-key
DEEPSEEK_API_KEY=your-deepseek-api-key
DEEPSEEK_BASE_URL=https://api.deepseek.com/v1
ADMIN_DEFAULT_PASSWORD=admin-initial-password
```

---

## 12. 目录结构（最终版）
 
```
CET_Vocab_Master/
├── nuxt.config.ts
├── package.json
├── tailwind.config.ts
├── tsconfig.json
├── .env
│
├── layouts/                      # 布局组件
│   ├── default.vue               # 用户端默认布局（顶部导航 + 侧边栏）
│   └── admin.vue                 # 管理端布局（左侧菜单 + 右侧内容区）
│
├── pages/                        # 用户端页面（Nuxt 文件路由自动注册）
│   ├── index.vue                 # 登录/注册页（同页 Tab 切换）
│   ├── read/
│   │   └── [id].vue              # 文章阅读页
│   └── generate.vue              # AI 短文定制页
│
├── admin/                        # 管理端页面
│   ├── index.vue                 # 数据看板 + 快捷入口
│   ├── articles.vue              # 文章管理
│   ├── vocabulary.vue            # 词库管理
│   └── templates.vue             # AI 模板管理
│
├── components/                   # 公共组件
│   ├── Sidebar.vue               # 侧边栏（文章目录）
│   ├── Popover.vue               # 查词气泡框
│   ├── ArticleReader.vue         # 文章阅读器（高亮 + 点击查词）
│   ├── StreamingText.vue         # 流式打字机渲染
│   ├── WordSelector.vue          # 词汇选择器
│   ├── TopicInput.vue            # 主题输入框 + 快捷标签
│   ├── LoginForm.vue             # 登录表单
│   ├── RegisterForm.vue          # 注册表单
│   └── admin/                    # 管理端专用组件
│       ├── ArticleForm.vue       # 文章创建/编辑表单
│       ├── VocabForm.vue         # 词汇创建/编辑表单
│       ├── TemplateForm.vue      # AI 模板创建/编辑表单
│       └── StatsCard.vue         # 统计数据卡片
│
├── composables/                  # 前端组合式函数
│   ├── useAuth.ts                # 认证状态管理
│   ├── useArticles.ts            # 文章数据获取
│   ├── useVocab.ts               # 词汇数据获取
│   ├── useStreaming.ts           # AI 流式生成（含 SSE 逻辑）
│   ├── useAudio.ts               # 发音播报（Web Speech API）
│   └── useProgress.ts            # 进度管理
│
├── stores/                       # Pinia 状态管理
│   ├── auth.ts                   # 用户认证状态
│   └── reading.ts                # 阅读状态
│
├── types/                        # TypeScript 类型定义
│   ├── user.ts                   # User 接口
│   ├── article.ts                # Article 接口
│   ├── vocabulary.ts             # Vocabulary 接口
│   ├── template.ts               # AITemplate 接口
│   ├── user-progress.ts          # UserProgress 接口
│   └── ai-generation-log.ts      # AIGenerationLog 接口
│
├── server/                       # 后端（Nuxt Server Routes）
│   ├── api/
│   │   ├── auth/
│   │   │   ├── register.post.ts  # 用户注册
│   │   │   ├── login.post.ts     # 登录（管理员/用户共用）
│   │   │   └── me.get.ts         # 获取当前用户
│   │   ├── articles/
│   │   │   ├── index.get.ts      # 文章列表
│   │   │   ├── index.post.ts     # 创建文章（管理员）
│   │   │   ├── [id].get.ts       # 文章详情
│   │   │   ├── [id].put.ts       # 更新文章（管理员）
│   │   │   └── [id].delete.ts    # 删除文章（管理员）
│   │   ├── vocab/
│   │   │   ├── index.get.ts      # 词汇列表
│   │   │   ├── index.post.ts     # 创建词条（管理员）
│   │   │   ├── [id].put.ts       # 更新词条（管理员）
│   │   │   └── [id].delete.ts    # 删除词条（管理员）
│   │   ├── templates/
│   │   │   ├── index.get.ts      # 模板列表
│   │   │   ├── index.post.ts     # 创建模板（管理员）
│   │   │   ├── [id].put.ts       # 更新模板（管理员）
│   │   │   └── [id].delete.ts    # 删除模板（管理员）
│   │   ├── progress/
│   │   │   ├── index.get.ts      # 获取用户进度
│   │   │   └── [articleId].put.ts # 更新熟记状态
│   │   ├── admin/
│   │   │   └── stats.get.ts      # 管理端统计数据
│   │   └── generate.post.ts      # AI 流式生成（SSE）
│   │
│   ├── middleware/
│   │   ├── auth.ts               # JWT 鉴权中间件（所有接口共用）
│   │   └── admin.ts              # 管理员角色鉴权中间件
│   │
│   ├── models/                   # Mongoose 数据模型
│   │   ├── User.ts
│   │   ├── Article.ts
│   │   ├── Vocabulary.ts
│   │   ├── AITemplate.ts
│   │   ├── UserProgress.ts
│   │   └── AIGenerationLog.ts
│   │
│   └── utils/
│       ├── db.ts                 # MongoDB 连接管理
│       └── jwt.ts                # JWT 签发/验证工具函数
│
├── doc/                          # 文档
│   ├── prd.md
│   ├── Acceptance Criteria.md
│   ├── stack.md
│   ├── architecture-decisions.md
│   └── dev.md                    # 本文档
```

---

## 13. 验收标准与开发对照

| 验收标准场景 | 涉及模块 | 关键文件 |
|---|---|---|
| 特性 1 场景一：注册成功 | 认证 | `pages/index.vue`、`server/api/auth/register.post.ts` |
| 特性 1 场景二：登录失败 | 认证 | `pages/index.vue`、`server/api/auth/login.post.ts` |
| 特性 1 场景三：状态保持 | 认证 | `composables/useAuth.ts`、`stores/auth.ts` |
| 特性 2 场景一：点击查词 | 阅读 | `components/ArticleReader.vue`、`components/Popover.vue`、`composables/useAudio.ts` |
| 特性 2 场景二：切换等级 | 阅读 | `components/Sidebar.vue`、`stores/reading.ts`、`composables/useVocab.ts` |
| 特性 2 场景三：关闭气泡框 | 阅读 | `components/Popover.vue` |
| 特性 2 场景四：高亮联动 | 阅读 | `components/ArticleReader.vue`、`composables/useVocab.ts` |
| 特性 3 场景一：从词库选词生成 | AI 生成 | `pages/generate.vue`、`components/WordSelector.vue`、`composables/useStreaming.ts` |
| 特性 3 场景二：从文章选词生成 | AI 生成 | `pages/read/[id].vue`、`pages/generate.vue` |
| 特性 3 场景三：数量不足校验 | AI 生成 | `pages/generate.vue`、`server/api/generate.post.ts` |
| 特性 3 场景四：数量上限校验 | AI 生成 | `pages/generate.vue`、`server/api/generate.post.ts` |
| 特性 3 场景五：生成失败处理 | AI 生成 | `composables/useStreaming.ts`、`components/StreamingText.vue` |
| 特性 4 场景一：跨设备同步 | 进度 | `server/api/progress/`、`composables/useProgress.ts` |
| 特性 4 场景二：取消熟记 | 进度 | `composables/useProgress.ts`、`components/Sidebar.vue` |
| 特性 5 场景一：管理员登录 | 管理端 | `admin/index.vue`、`server/middleware/admin.ts` |
| 特性 5 场景二：上传文章 | 管理端 | `admin/articles.vue`、`components/admin/ArticleForm.vue` |
| 特性 5 场景三：编辑文章 | 管理端 | 同上 |
| 特性 5 场景四：删除文章 | 管理端 | 同上 |
| 特性 5 场景五：词库管理 | 管理端 | `admin/vocabulary.vue`、`components/admin/VocabForm.vue` |
| 特性 5 场景六：模板管理 | 管理端 | `admin/templates.vue`、`components/admin/TemplateForm.vue` |
