# 平台化改造设计：接入 Cloudflare DB 的轻量社交游戏平台（feature/database）

- 日期：2026-07-31
- 状态：已批准（待 writing-plans 细化实施）
- 范围：9 个 H5 游戏 + game-list 总入口 + 新建后端 game-api

## 1. 背景与目标

当前 9 个游戏均为纯静态、零依赖、localStorage 存储、无后端、无登录。各游戏分数只存在本机，没有跨用户/跨设备概念；game-list 目前只是分享卡片生成器，不是入口。

本改造把它们升级为一个带 Cloudflare 后端的轻量社交平台，提供：

- 统一身份（自定义登录：昵称 + 密码 + 头像，存 D1，头像存 R2）
- 跨游戏单点登录（SSO，URL 传 token）
- 在线状态「谁在玩」（KV 心跳）
- 天梯榜（每游戏前 10 + game-list 总入口各游戏最强者）
- 分享整合（game-list 已有分享卡，挂上用户战绩）

### 关键约束（已澄清）
- 微信资质：用户为**个人订阅号，无法认证**，因此无法自动获取微信头像/昵称。登录采用**纯自定义登录 MVP**（昵称 + 密码 + 头像），未来若有认证服务号可平滑升级为公众号 OAuth（snsapi_userinfo）。
- 数据库：**D1（SQLite）+ KV 混合** —— D1 存用户/分数/天梯，KV 存在线心跳（高频写、TTL 自动过期）。头像图片存 **R2** 对象存储。
- 后端：**独立共享 Worker 后端**（新建仓库 game-api），9 个游戏前端都 fetch 它。
- 天梯榜口径：**每游戏各自前 10 + game-list 总入口展示各游戏最强者（榜首）**。
- SSO 机制：**URL 传 token（MVP）**，token 短期有效降低泄露风险。
- 头像：**存于 R2**，上传时**限制图片大小**（类型白名单 + 文件大小上限 + 可选尺寸上限）。

## 2. 范围（In / Out）

**In：**
- 自定义账号体系（注册/登录/改资料，含密码哈希）
- 头像上传与存储（R2，限大小/类型）
- 跨游戏 SSO（game-list 登录后跳游戏带 token）
- 在线状态（每游戏在线人数 + 头像；总入口全局在线）
- 天梯榜（每游戏前 10；总入口各游戏最强者）
- 成绩上报与展示（各游戏定义自己的 score）
- 分享整合（game-list 分享卡带用户战绩）
- 后端 game-api（Worker + D1 + KV + R2 + API + 前端 SDK）

**Out（YAGNI）：**
- 微信 OAuth 自动头像（个人号不可认证）
- 支付、聊天、好友系统、公会
- WebSocket 实时推送（用轮询）
- 多端同步（本机存档仍保留，云端仅存社交数据）
- 头像服务端压缩/裁剪（MVP 仅校验大小与类型，不做服务端图像处理）

## 3. 总体架构

```
┌─────────────┐   登录/心跳/成绩/榜单   ┌──────────────────────┐
│  game-list  │ ──────────────────────▶ │   game-api (Worker)  │
│ (总入口)    │ ◀────────────────────── │  + D1 + KV + R2      │
└──────┬──────┘    ?token= 跳转          └──────────────────────┘
       │  SSO
       ▼
┌─────────────┐   引入 sdk.js            ┌──────────────────────┐
│ 各游戏 H5   │ ──────────────────────▶ │  GamePlatform SDK    │
│ (*.pages.dev)│                         │  (托管在 game-api 域) │
└─────────────┘                          └──────────────────────┘
```

- **独立后端 `game-api`**（新建仓库，Cloudflare Worker + D1 + KV + R2），统一托管所有 API 与头像对象存储。
- **前端共享 SDK `sdk.js`**：部署在 `game-api` 域名下，各游戏用 `<script src="https://<api-domain>/sdk.js">` 引入（零构建、零依赖，契合现状）。负责：读 URL token、localStorage 持久化、心跳、提交成绩、拉榜、登录/注册/资料/头像上传。
- **各游戏**：引入 sdk.js，加登录条、在线条、天梯面板、成绩提交钩子。
- **game-list**：升级为总入口（登录 + 在线玩家 + 各游戏最强者 + 分享）。

## 4. 数据模型

### D1
- `users(id INTEGER PK, nickname TEXT UNIQUE, password_hash TEXT, avatar_url TEXT, created_at INTEGER)`
  - `password_hash`：注册时用 **Web Crypto（PBKDF2 + 随机盐）** 哈希存储，绝不存明文，登录时校验。
  - `avatar_url`：指向 R2 公开访问地址（`https://<r2-public>/avatars/<userId>.<ext>`）。缺省用本地默认头像（不写 R2）。
- `scores(id INTEGER PK, user_id INTEGER, game_id TEXT, score REAL, meta TEXT(JSON), created_at INTEGER)`
  - 索引：`CREATE INDEX idx_scores_game ON scores(game_id, score)`
  - `meta` 存该局上下文（难度、时长等），JSON 字符串。

> token 用**签名 JWT**（HS256，后端密钥），不落库，后端校验签名即可，避免 sessions 表。

### KV（在线状态）
- `online:<game_id>:<user_id>` = `{nickname, avatar, ts}`，TTL 60s
- `online:global:<user_id>` = `{game_id, ts}`，TTL 60s（供总入口全局在线）

### R2（头像）
- Bucket：`avatars`，键 `avatars/<userId>.<ext>`（ext 由上传类型决定：jpg/png/webp/gif）。
- 公开访问：通过 R2 公共桶域名或自定义域 `https://<r2-public>/avatars/<userId>.<ext>`。
- **大小/类型限制**：上传接口硬性拒绝 > 2MB 或非白名单类型；可选约束最长边 ≤ 512px（前端提示，后端不强制裁剪）。

## 5. 后端 API（Worker 路由）

| 方法 | 路径 | 说明 |
|---|---|---|
| POST | `/api/auth/register` | `{nickname, password, avatar?}` → 昵称查重，哈希密码建 user，返 JWT（409 若昵称已存在）；avatar 可选，缺省用默认头像 |
| POST | `/api/auth/login` | `{nickname, password}` → 校验密码（PBKDF2 比对），返 JWT（401 若错） |
| GET | `/api/auth/me` | 返当前用户资料（Bearer） |
| PUT | `/api/auth/profile` | 改昵称/头像 URL/密码 |
| POST | `/api/avatar` | `multipart/form-data` 上传头像至 R2；**前端 + 后端双重校验**：类型白名单(jpg/png/webp/gif)、文件大小 ≤ 2MB，可选最长边 ≤ 512px；返回 R2 公开 URL（需 token） |
| POST | `/api/scores` | `{game_id, score, meta}` 提交成绩（需 token） |
| GET | `/api/leaderboard/:game_id?limit=10` | 前 10（按 game registry 的 direction 排序） |
| POST | `/api/online/heartbeat` | `{game_id}` 刷新在线（KV TTL） |
| GET | `/api/online/:game_id` | 本游戏在线列表 |
| GET | `/api/online/global` | 全局在线 |

排序方向因游戏而异：后端维护 **game registry**（`game_id → {label, metric, direction: asc|desc}`），排序按 direction。MVP 硬编码 registry。

**认证**：除 register/login 外所有接口需 `Authorization: Bearer <jwt>`；JWT 校验失败返 401。

**密码哈希**：Worker 内用 `crypto.subtle` 的 PBKDF2（`SHA-256`，随机 16 字节盐，迭代 ≥ 100k），存 `salt:hash` 格式。

**头像上传限制（硬性）**：
- 仅接受 `image/jpeg`、`image/png`、`image/webp`、`image/gif`。
- 请求体 `Content-Length` 超过 2MB 直接拒收（413）；超类型白名单返 400。
- 校验通过后写入 R2（`avatars/<userId>.<ext>`，覆盖同名即更新头像），返回公开 URL；调用方再用 `PUT /api/auth/profile` 写入 `users.avatar_url`。

## 6. 前端 SDK 与 SSO

`window.GamePlatform` 暴露：
- `init()`：读 URL `?token=`，存 `localStorage('gp_token')`；若无则尝试读 localStorage 已有 token。
- `register(nickname, password, avatar?)` → `/api/auth/register`
- `login(nickname, password)` → `/api/auth/login`
- `uploadAvatar(file)` → `/api/avatar`（**上传前前端校验大小 ≤ 2MB 与类型**，不合规直接提示，避免无效请求）
- `getToken()` / `getUser()`
- `heartbeat(gameId)`：每 30s 调 `/api/online/heartbeat`
- `submitScore(gameId, score, meta)`
- `getLeaderboard(gameId, limit)`
- `getOnline(gameId)` / `getOnlineGlobal()`

**SSO 流程**：用户在 game-list 登录 → 拿到 JWT → 点击某游戏时跳转拼 `?token=<jwt>` → 目标游戏 `init()` 读参存入自己 localStorage → 之后自动带 token 调 API。跳转无需再输密码。JWT 短期有效（如 7 天）降低 URL 泄露风险。

**SDK 分发**：sdk.js 作为静态资源托管在 game-api 域名（如 `https://<api-domain>/sdk.js`），各游戏 `<script>` 引入；SDK 内 `API_BASE` 通过 `<meta name="api-base">` 或脚本 URL 同源推断，避免硬编码。

## 7. 各游戏改造（统一模式）

每个 `index.html` 加：
- **登录条**：未登录显示「登录/注册」按钮 → 弹昵称 + 密码 + 头像上传（**限制 ≤ 2MB、类型白名单，前端即时提示**）；已登录显示头像 + 昵称 + 退出。
- **在线条**：「X 人在玩」+ 在线用户头像（每 15s 刷新，每 30s 心跳）。
- **天梯面板**：本游戏前 10 列表（昵称 + 分数 + 头像）。
- **成绩提交钩子**：游戏结束/结算时调 `submitScore(gameId, computedScore, meta)`。

各游戏自定义 score 与 direction（注册进 game registry）：
| game_id | 指标 | direction |
|---|---|---|
| mine-sweeper | 难度权重 + 用时（秒） | asc（越快越好） |
| greedy-snake | 蛇长度 | desc |
| flop | 配对用时（秒） | asc |
| holy-grail | 最高圣杯连击数 | desc |
| life-restart-simulator | 结局评分 | desc |
| spend-money-for-musk | 净资产/成就分 | desc |
| human-bench | 综合分 | desc |
| lunch | 娱乐向，MVP 可不参与天梯（或记「抽中次数」） | — |

## 8. game-list 总入口改造

- 加登录/资料入口（含头像上传，受同样大小/类型限制）。
- 顶部「在线玩家」横幅（全局在线头像，调 `/api/online/global`）。
- 每个游戏卡片下显示「最强者：昵称 + 分数」（调 `/api/leaderboard/:game_id?limit=1`）。
- 分享：分享卡带用户战绩 / 总入口链接（复用现有 Canvas 分享卡 + 二维码逻辑）。

## 9. 在线状态实时性

- 前端每 30s `heartbeat`，KV TTL 60s（双倍余量防抖动）。
- 在线列表每 15s 轮询刷新。
- **不引入 WebSocket**（轮询足够，YAGNI）。

## 10. 分支与部署策略

- 新建后端仓库 `game-api`（建 `feature/database` 分支）：含 `wrangler.toml`（D1 + KV + **R2** 绑定）、Worker 入口、路由、前端 `sdk.js`。
- game-list + 8 个参与游戏仓库均建 `feature/database` 分支，引入 sdk.js、加登录/在线/天梯。
- 部署：`game-api` 独立 Worker（自定义域或 `*.workers.dev`）；R2 桶 `avatars` 开启公共访问（或绑自定义域）以直链加载头像；各游戏 Pages 加 `API_BASE` 环境变量指向该域。
- 注意：URL 传 token 含 JWT，务必 HTTPS + 短期有效；跳转后 token 立即存入 localStorage 并从 URL 清除（`history.replaceState`）。

## 11. 分阶段实施（writing-plans 细化）

- **Phase 1**：后端骨架（D1 + KV + **R2** + Worker + auth/scores/online/avatar API）+ 前端 SDK + 密码哈希 + 头像大小限制。
- **Phase 2**：game-list 总入口改造（登录 + 在线 + 最强者 + 分享 + 头像上传）。
- **Phase 3**：各游戏接入 SDK（登录条 + 在线条 + 天梯 + 成绩提交 + 头像上传）。
- **Phase 4**：联调与本地验证（wrangler dev + 本地前端 + R2 本地模拟）。

## 12. 成功标准

- 在 game-list 注册账号（昵称 + 密码 + 头像）后，跳转到任意游戏无需重新登录即显示已登录态与头像。
- 头像上传被限制为 ≤ 2MB、白名单类型；超限前端即时提示、后端硬性拒收。
- 某游戏进行时，该游戏在线条显示正确人数；game-list 全局在线横幅显示该用户。
- 游戏结束后成绩进入该游戏天梯前 10；game-list 对应卡片显示最强者。
- 密码不以明文存储；JWT 过期后需重新登录。

## 13. 风险与未决

- 个人号无法微信 OAuth：已用自定义登录兜底，未来升级路径明确。
- 跨子域 SSO 依赖 URL token：已用短期 JWT + 跳转后清除缓解。
- `lunch` 是否参与天梯：MVP 可不参与，待定。
- R2 公开访问域名配置（自定义域或 `*.r2.dev` 公共桶），需确保头像 URL 可被各游戏跨域加载。
