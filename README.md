# 游戏合集（Game List）

纯前端游戏合集分享卡片页面：汇总所有游戏，一键生成分享卡片。

## 特性

- **纯静态**：单文件 `index.html`（内联 CSS/JS），零依赖、无构建步骤。
- **数据本地**：游戏列表在页面内 `GAMES.categories` 数组中维护。
- **移动端适配**：针对触屏与微信 webview 优化。
- **分享卡片**：可生成合辑分享图（二维码需联网）。

## 本地运行

```sh
cd game-list
python3 -m http.server 8000
# 浏览器打开 http://localhost:8000
```

> 分享卡片的二维码依赖 `api.qrserver.com`，必须经 `http(s)` 来源加载，请用本地服务器方式打开，不要直接 `file://` 打开。

## 文件结构

```
game-list/
└── index.html   # 单文件，含全部 HTML/CSS/JS
```

## 维护

编辑 `GAMES.categories` 数组可添加 / 更新游戏。三个分类：模拟体验、休闲娱乐、玄学趣玩。跳过非游戏项目（如 `game-api`、`game-list` 自身）。

## 部署

已部署至 Cloudflare Pages：`game-list-arv.pages.dev`
