# 游戏合集分享卡片生成器（Game List）

纯前端单机工具：汇总所有游戏，一键生成分享卡片页面。

## 单机版特性

- **纯静态**：仅 `index.html`（内联 CSS/JS），零依赖、无构建步骤。
- **无需联网**：卡片渲染逻辑在浏览器本地运行（分享卡二维码需联网）。
- **数据本地**：游戏列表在页面内 `GAMES.categories` 数组中维护。
- **即开即玩**：双击 `index.html` 即可运行。

## 本地运行

```sh
cd game-list
python3 -m http.server 8000
# 浏览器打开 http://localhost:8000
```

> 分享卡片二维码依赖在线服务，请用本地服务器方式打开，不要直接 `file://` 打开。

## 文件结构

```
game-list/
└── index.html   # 单文件，含全部 HTML/CSS/JS
```

## 维护

编辑 `GAMES.categories` 数组可添加 / 更新游戏。三个分类：模拟体验、休闲娱乐、玄学趣玩。跳过非游戏项目（如 `agent-comparison-2026`、`game-list` 自身）。

## 部署

可部署到 Cloudflare Pages。

## 版本

当前分支：`release/1.0.0`
