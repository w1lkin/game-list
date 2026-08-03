# 实施计划：接入 Cloudflare DB 的轻量社交游戏平台（feature/database）

- 日期：2026-07-31
- 依据：specs/2026-07-31-platform-database-design.md（已批准）
- 范围：新建后端 `game-api` + 改造 `game-list` 总入口 + 接入 8 个参与游戏
- 分支约定：所有仓库在 `main` 基础上新建并切到 `feature/database` 分支开发。

---

## 0. 前置准备（环境检查，任何 Phase 前执行一次）

**目标**：确认本机可构建/部署 Cloudflare 资源。

- [ ] 检查 `wrangler`：`npx wrangler --version`（无则 `npm i -g wrangler`）。
- [ ] 登录：`npx wrangler login`（浏览器授权），确认 `npx wrangler whoami` 返回账号。
- [ ] 确认可创建资源：D1 / KV / R2 配额（免费额度足够 MVP）。
- [ ] 选定 `game-api` 部署域名：自定义域或 `<sub>.workers.dev`；记录为 `API_BASE`（例：`https://game-api.<user>.workers.dev`）。
- [ ] 选定 R2 公开访问方式：桶 `avatars` 开启公共访问（得 `*.r2.dev` 公共域名）或绑自定义域；记录为 `R2_PUBLIC`。

> 若环境未就绪，先完成登录与配额确认；代码骨架可先行编写（不依赖部署）。

---

## Phase 1：后端骨架 + 前端 SDK

新建仓库 `game-api`（本地目录 `/Users/welkin/Code/game/game-api`，`git init`，建 `feature/database` 分支）。

### 1.1 项目脚手架
- [ ] `wrangler.toml`：绑定
  - `[[d1_databases]]`：`binding = "DB"`，`database_name = "game_platform"`，`database_id`（占位，`wrangler d1 create` 后回填）。
  - `[[kv_namespaces]]`：`binding = "KV"`，`id`（占位，`wrangler kv namespace create` 后回填）。
  - `[[r2_buckets]]`：`binding = "R2"`，`bucket_name = "avatars"`，`public = true`（或自定义域）。
  - `vars`：`JWT_SECRET`（部署时用 `wrangler secret put JWT_SECRET` 注入，**不写明文**）、`R2_PUBLIC`、`API_BASE`。
  - `assets`：托管 `sdk.js` 等静态资源（或放 `/public` 由 Worker 直链返回）。
- [ ] `package.json`：`name: game-api`，`type: module`，`devDependency: wrangler`。
- [ ] `.gitignore`：`node_modules/`、`.wrangler/`、`.dev.vars`。

### 1.2 D1 schema
文件 `migrations/0001_init.sql`：
- [ ] `CREATE TABLE users(id INTEGER PRIMARY KEY AUTOINCREMENT, nickname TEXT UNIQUE NOT NULL, password_hash TEXT NOT NULL, avatar_url TEXT, created_at INTEGER NOT NULL);`
- [ ] `CREATE TABLE scores(id INTEGER PRIMARY KEY AUTOINCREMENT, user_id INTEGER NOT NULL, game_id TEXT NOT NULL, score REAL NOT NULL, meta TEXT, created_at INTEGER NOT NULL, FOREIGN KEY(user_id) REFERENCES users(id));`
- [ ] `CREATE INDEX idx_scores_game ON scores(game_id, score);`
- [ ] 执行：`npx wrangler d1 execute game_platform --local --file=./migrations/0001_init.sql`（本地），并准备 `--remote` 版本用于部署。

### 1.3 密码哈希工具 `src/auth.js`
- [ ] `hashPassword(pw)`：PBKDF2-SHA256，随机 16B 盐，迭代 ≥ 100000，返回 `salt:hash`（hex）。
- [ ] `verifyPassword(pw, stored)`：重算比对，恒定时间比较。
- [ ] 单元测试（Node）：`node --test tests/auth.test.js`（纯 Web Crypto，可在 Node ≥ 20 运行）。

### 1.4 JWT `src/jwt.js`
- [ ] `sign(payload)` / `verify(token)`：HS256，`crypto.subtle` HMAC，密钥来自 `env.JWT_SECRET`；payload 含 `sub=user_id`、`nick`、`exp`（7 天）。
- [ ] 单元测试：`tests/jwt.test.js`（sign→verify round-trip + 篡改失败）。

### 1.5 game registry `src/registry.js`
- [ ] 硬编码 `GAMES = { mine-sweeper:{label,direction:'asc',metric}, greedy-snake:{...desc}, flop:{asc}, holy-grail:{desc}, life-restart-simulator:{desc}, spend-money-for-musk:{desc}, human-bench:{desc}, lunch:{skip:true} }`（与 spec 第 6 节一致）。

### 1.6 Worker 路由 `src/index.js`
- [ ] 中间件：除 `/api/auth/register`、`/api/auth/login`、`/sdk.js` 外，校验 `Authorization: Bearer`；失败 401。
- [ ] `POST /api/auth/register`：昵称查重（409）、哈希密码、建 user、签 JWT 返回。
- [ ] `POST /api/auth/login`：查 user、校验密码（401）、签 JWT。
- [ ] `GET /api/auth/me`：返回 user（脱敏，不含 password_hash）。
- [ ] `PUT /api/auth/profile`：改昵称/头像 URL/密码（密码重哈希）。
- [ ] `POST /api/scores`：`{game_id, score, meta}` 校验 registry、写 scores、返回插入结果。
- [ ] `GET /api/leaderboard/:game_id`：`ORDER BY score <direction> LIMIT ?`，JOIN users 取昵称/头像；`lunch` 返回 204/空。
- [ ] `POST /api/online/heartbeat`：`{game_id}` 写 `KV online:<game_id>:<uid>` 与 `online:global:<uid>`（TTL 60）。
- [ ] `GET /api/online/:game_id` / `/api/online/global`：列举 KV 前缀、解析、去重。
- [ ] `POST /api/avatar`：multipart 解析；**双重校验**：`Content-Type` 白名单(jpg/png/webp/gif) + `Content-Length ≤ 2MB`（超 413，超类型 400）；写 `R2.put('avatars/<uid>.<ext>')`；返回 `env.R2_PUBLIC/avatars/<uid>.<ext>`。
- [ ] `GET /sdk.js` 或静态资源：从 `assets`/本地文件返回 SDK 文本（`text/javascript`）。

### 1.7 前端 SDK `public/sdk.js`
- [ ] `window.GamePlatform`：
  - `init()`：读 `?token=`→存 `localStorage('gp_token')`→`history.replaceState` 清 URL；否则读 localStorage。
  - `register(nickname, password, avatar?)` / `login(nickname, password)` / `uploadAvatar(file)`（**前端前置校验 ≤2MB + 类型**，不合规则 reject 不发包）。
  - `getUser()` / `getToken()` / `logout()`。
  - `heartbeat(gameId)`（30s 定时器封装 `startHeartbeat`），`submitScore(gameId, score, meta)`，`getLeaderboard(gameId, limit)`，`getOnline(gameId)`，`getOnlineGlobal()`。
  - `API_BASE`：从 `<meta name="api-base">` 取，缺省用引入脚本所在 origin。
  - 所有请求带 `Authorization: Bearer` 与 `Content-Type: application/json`（multipart 除外）。
- [ ] 在本地起 http server 用一页测最小闭环（无 UI）。

### 1.8 Phase 1 验证
- [ ] `node --test tests/*.test.js` 全绿（auth/jwt）。
- [ ] `npx wrangler dev` 起本地 Worker；用 `curl` 跑通：register→login→me→profile→avatar（小图成功、>2MB 返 413、超类型返 400）→scores→leaderboard→heartbeat→online。
- [ ] `curl` 拉 `/sdk.js` 返回 JS 文本。
- [ ] 部署：`wrangler deploy`（远程 D1/KV/R2 已 `create` 并回填 id），记录 `API_BASE`、`R2_PUBLIC`。

---

## Phase 2：game-list 总入口改造

仓库 `game-list`，切 `feature/database` 分支。

- [ ] 引入 `<script src="${API_BASE}/sdk.js">`，加 `<meta name="api-base">`。
- [ ] 顶部「在线玩家」横幅：`getOnlineGlobal()` 每 15s 刷新头像。
- [ ] 登录/资料入口：未登录→登录/注册弹层（昵称+密码+头像上传，受 ≤2MB/类型限制）；已登录→头像+昵称+退出+改资料。
- [ ] 游戏卡片：每个卡片下加「最强者：昵称 + 分数」，`getLeaderboard(gameId, 1)` 填充（lunch 跳过）。
- [ ] 跳转：点游戏卡片时用 `?token=${getToken()}` 拼接到游戏 URL。
- [ ] 分享：分享卡带用户战绩 / 总入口链接（复用现有 Canvas+二维码逻辑）。
- [ ] 验证：本地 `python3 -m http.server` 打开，注册→见在线横幅→见各游戏最强者→点游戏带 token 跳转。

---

## Phase 3：各游戏接入 SDK

8 个仓库（flop、greedy-snake、holy-grail、life-restart-simulator、lunch、mine-sweeper、spend-money-for-musk、human-bench），各切 `feature/database` 分支。

每个游戏的**统一任务**（按各项目结构落地，单文件 `index.html` 直接内联调度；greedy-snake/mine-sweeper/spend-money-for-musk 用其 js 模块接入）：
- [ ] 引入 sdk.js（`API_BASE` 走 env 或 meta）。
- [ ] `GamePlatform.init()` 在页面加载即调用。
- [ ] **登录条**：未登录按钮→弹登录/注册（含头像上传限制）；已登录显示头像+昵称+退出。
- [ ] **在线条**：`startHeartbeat(gameId)` + `getOnline(gameId)` 每 15s 刷新「X 人在玩」+ 头像。
- [ ] **天梯面板**：`getLeaderboard(gameId, 10)` 渲染前 10（昵称+分数+头像）。
- [ ] **成绩提交钩子**：游戏结束/结算逻辑里调 `submitScore(gameId, computedScore, meta)`，game_id 与 spec 第 6 节一致；lunch 可跳过或不参与。
- [ ] 各游戏 score 计算与 direction（见 spec 表）。

> 注意：greedy-snake / mine-sweeper / spend-money-for-musk 已有模块化 js，把 SDK 调用挂到其状态机/结算点；flop / holy-grail / life-restart-simulator 为单文件 index.html，直接内联。

- [ ] 验证（逐游戏）：本地 http server 打开→登录→玩一局→成绩进天梯→在线条显示自己。

---

## Phase 4：联调与本地验证

- [ ] game-api 远程可访问（API_BASE 生效），D1/KV/R2 远程数据可读写。
- [ ] 全链路：game-list 注册→跳某游戏（URL token）→游戏内已登录+头像→玩→成绩进该游戏前 10→回 game-list 见该游戏最强者。
- [ ] 头像：上传 1.9MB jpg 成功显示；上传 2.1MB 返 413 且前端提示；上传 txt 返 400 且前端提示。
- [ ] 在线：两浏览器标签（或设备）各登录，互相可见在线头像。
- [ ] 密码：DB 中 `password_hash` 为 `salt:hash`，无明文；JWT 过期（改短 exp 测）后需重登。
- [ ] 回归：各游戏本机 localStorage 存档仍正常（不破坏既有玩法）。

---

## 提交与合并节奏

- 每个 Phase 完成后在对应仓库 `commit`；Phase 1/2/3 各自分支独立推进。
- 全部验证通过后，由各仓库 `feature/database` 向 `main` 合回（fast-forward/merge），并视情况推送、打 `v1.1.0` tag。
- `game-api` 单独仓库，部署后再合 main。

## 风险与回滚

- R2 公开访问若配置失败：头像回退为默认本地头像（不影响主流程）。
- URL token 泄露：JWT 7 天短期 + 跳转即清 URL 缓解；上线可升级 code 换 token（spec 已预留）。
- 任一游戏接入失败：其余不受影响，可单独回滚该分支。
