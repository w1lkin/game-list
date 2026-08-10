# CHANGELOG

## [2.0.0] - 2026-08-10

### Docs
- 新建 `AGENTS.md`（项目架构与 AI 协作指南）
- 更新 `README.md`，统一格式；分类从三个扩展为四个（加入脑力挑战）

---

## [1.0.0] - 2026-07

### Added
- 游戏合集初始版本：分享卡片生成器
- 单文件 `index.html`（内联 CSS/JS），零依赖
- Canvas 绘制 1200×1600 合辑分享图
- 游戏列表在 `GAMES.categories` 数组中维护
- 三个分类：模拟体验、休闲娱乐、玄学趣玩

### Changed
- 接入 GamePlatform 登录门
- 统一命名「小游戏」+ 底部 tab（游戏/我的）
- 门户页天梯榜按难度分组显示
- 域名改回 Cloudflare Pages 默认域名 `game-list-arv.pages.dev`
- 部署至 Cloudflare Pages
