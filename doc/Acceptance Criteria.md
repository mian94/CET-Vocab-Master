# 基于 Gherkin 语法的验收标准 (Acceptance Criteria)

---

## 特性 1: 用户注册与登录 (Registration & Login)

```gherkin
# 场景一：新用户注册成功
Scenario: New user registers successfully
  Given 用户打开注册页面
  When 用户输入用户名 "testuser"、手机号 "13800001111"、密码 "Passw0rd!"
  And 用户点击 "注册" 按钮
  Then 系统应创建账号并自动登录
  And 页面应跳转至阅读首页

# 场景二：登录失败提示
Scenario: Login fails with wrong password
  Given 用户打开登录页面
  When 用户输入用户名 "testuser" 和错误密码 "wrongpass"
  And 用户点击 "登录" 按钮
  Then 系统应提示 "用户名或密码错误"

# 场景三：登录状态保持
Scenario: Login state persists after page refresh
  Given 用户已成功登录
  When 用户刷新浏览器页面
  Then 用户应仍处于登录状态，无需重新输入密码
  And 阅读进度和个人数据应正常加载
```

---

## 特性 2: 核心阅读与交互 (Interactive Reading)

```gherkin
# 场景一：点击高亮单词查看释义与发音
Scenario: Click a highlighted word to view definition and pronunciation
  Given 用户已成功登录并进入一篇四级短文阅读页面
  When 用户点击文章中高亮的单词 "substitute"
  Then 页面应当在单词上方弹出一个气泡框
  And 气泡框内应展示该单词的音标以及中文释义 "n. 代替者 v. 代替"
  And 气泡框内应包含一个发音图标，点击后可播放该单词的英/美式读音

# 场景二：切换词库级别
Scenario: Switch vocabulary level between CET-4 and CET-6
  Given 用户当前正在阅读 "四级" 目录下的文章
  When 用户点击顶部导航栏的 "六级" 切换按钮
  Then 侧边栏的短文目录应当立即更新为六级高频词文章列表
  And 当前视图加载六级目录下的第一篇文章

# 场景三：点击文章外部区域关闭气泡框
Scenario: Close popover by clicking outside
  Given 用户已点击一个高亮单词，气泡框处于展开状态
  When 用户点击文章中气泡框以外的空白区域
  Then 气泡框应当关闭

# 场景四：切换等级后高亮词汇联动更新
Scenario: Highlighted words update when switching vocabulary level
  Given 用户正在阅读一篇同时包含四级和六级词汇的文章
  And 当前处于 "四级" 模式，文章中 "substitute" 等四级词汇被高亮
  When 用户切换至 "六级" 模式
  Then 文章中 "substitute" 等仅属于四级的词汇应取消高亮
  And 文章中属于六级词库的词汇应变为高亮状态
```

---

## 特性 3: AI 短文定制生成 (AI Custom Story Generation)

```gherkin
# 场景一：从词汇列表选择单词并成功生成 AI 短文
Scenario: Generate an AI story by selecting words from vocabulary list
  Given 用户进入 "AI定制" 页面
  When 用户在词汇列表中勾选了 "sustainable", "innovation", "advocate"
  And 用户在主题输入框中输入 "科技与环保"
  And 用户点击 "开始生成" 按钮
  Then 系统应当开始以流式打字机效果逐字展示新生成的短文
  And 生成的短文中必须包含上述勾选的三个单词，并且这三个单词需被高亮标记
  And 该文章会自动添加至用户的侧边栏 "我的定制" 目录中

# 场景二：从文章中选择单词并成功生成 AI 短文
Scenario: Generate an AI story by selecting words from an article
  Given 用户正在阅读一篇四级短文
  When 用户在文章中勾选了 "substitute", "efficient", "perspective" 等单词
  And 用户点击 "用这些词生成" 按钮，进入 "AI定制" 页面
  And 用户在主题输入框中输入 "校园生活"
  And 用户点击 "开始生成" 按钮
  Then 系统应当以流式打字机效果展示新生成的短文
  And 生成的短文中必须包含上述勾选的单词，并被高亮标记

# 场景三：选择单词数量不足时的校验
Scenario: Validate minimum number of selected words
  Given 用户在 "AI定制" 页面
  When 用户仅勾选了 1 个单词，并输入了主题
  And 用户点击 "开始生成" 按钮
  Then 系统应当拦截请求，并弹出错误提示 "请至少选择 5 个单词以确保生成质量"

# 场景四：选择单词数量超过上限时的校验
Scenario: Validate maximum number of selected words
  Given 用户在 "AI定制" 页面
  When 用户勾选了 11 个单词，并输入了主题
  And 用户点击 "开始生成" 按钮
  Then 系统应当拦截请求，并弹出错误提示提示用户最多选择 10 个单词

# 场景五：AI 生成失败时的错误处理
Scenario: Handle AI generation failure gracefully
  Given 用户在 "AI定制" 页面
  And 用户已勾选 5 个单词并输入了主题
  When 用户点击 "开始生成" 按钮
  And AI 接口返回超时或网络错误
  Then 系统应当停止打字机效果
  And 显示错误提示 "生成失败，请稍后重试"
```

---

## 特性 4: 进度同步与云端持久化 (Progress Sync & Persistence)

```gherkin
# 场景一：标记文章为已熟记并在跨设备登录时同步
Scenario: Mark a story as mastered and sync across devices
  Given 用户在电脑端登录账号，正在阅读编号为 #005 的短文
  When 用户点击文章底部的 "已熟记" 按钮
  Then 侧边栏目录中 #005 文章旁应当显示绿色的 "已勾选" 状态图标
  And 云端 MongoDB 数据库中该用户的进度记录应同步更新
  When 用户随后在手机端浏览器登录同一个账号
  Then 手机端的侧边栏目录中 #005 文章应当同样显示为 "已熟记" 状态

# 场景二：取消已熟记标记
Scenario: Unmark a story from mastered status
  Given 用户正在阅读一篇已标记为 "已熟记" 的文章
  When 用户再次点击 "已熟记" 按钮（或显示为 "取消熟记"）
  Then 该文章的 "已熟记" 状态应被取消
  And 侧边栏目录中该文章的绿色勾选图标应消失
  And 云端数据库中该文章的进度记录应同步更新为未熟记
```

---

## 特性 5: 管理端 (Admin Panel)

```gherkin
# 场景一：管理员通过独立入口登录
Scenario: Admin logs in through independent URL path
  Given 管理员打开管理端独立 URL 路径 "/admin"
  When 管理员输入管理员账号和密码
  And 点击 "登录" 按钮
  Then 系统应验证身份并跳转至管理端后台首页
  And 普通用户使用普通用户账号访问 "/admin" 应被拒绝，提示无权限

# 场景二：管理员上传预置文章并在用户端实时可见
Scenario: Admin uploads an article and it appears on user side in real time
  Given 管理员已登录管理端，进入 "文章管理" 页面
  When 管理员上传一篇新文章，设置等级为 "四级"，主题标签为 "#科技发展"
  And 管理员标记文章中需要高亮的目标词汇
  And 管理员点击 "发布" 按钮
  Then 普通用户端在不刷新页面的情况下
  And 用户切换至 "四级" 模式时，侧边栏目录中应能看到该新文章
  And 文章中被标记的词汇应以高亮形式展示

# 场景三：管理员编辑文章后用户端实时同步
Scenario: Admin edits an article and changes reflect on user side in real time
  Given 管理员已登录管理端，进入 "文章管理" 页面
  And 用户端当前正在阅读一篇四级短文 #003
  When 管理员编辑 #003 文章的内容或修改高亮词汇标记
  And 管理员点击 "保存" 按钮
  Then 用户端正在阅读该文章的用户应看到内容实时更新
  And 高亮词汇应同步更新为管理员最新标记的词汇

# 场景四：管理员删除文章后用户端实时同步
Scenario: Admin deletes an article and it disappears from user side in real time
  Given 管理员已登录管理端，进入 "文章管理" 页面
  When 管理员删除一篇四级短文 #007
  And 管理员确认删除操作
  Then 用户端侧边栏目录中 #007 文章应实时消失
  And 若用户当前正在阅读 #007，应提示 "该文章已被移除"

# 场景五：管理员管理词库词条
Scenario: Admin manages vocabulary entries
  Given 管理员已登录管理端，进入 "词库管理" 页面
  When 管理员新增一个词条，填写单词 "sustainable"、音标、释义、等级 "六级"
  And 管理员点击 "保存" 按钮
  Then 用户端在六级模式下，文章中出现 "sustainable" 时应被高亮标记
  And 用户点击该单词时，气泡框应展示管理员填写的音标和释义

# 场景六：管理员管理 AI 模板文章
Scenario: Admin manages AI template articles
  Given 管理员已登录管理端，进入 "AI模板管理" 页面
  When 管理员上传一篇模板文章，设置主题标签为 "#环境保护"
  And 管理员点击 "保存" 按钮
  Then 当用户在 AI 定制页面选择 "环境保护" 相关主题生成短文时
  And 系统应参考该模板文章的风格和结构进行生成
```
