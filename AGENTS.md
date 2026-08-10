# 游戏合集

## 项目概览

游戏合集分享卡片生成器：单文件静态 HTML，Canvas 绘制 1200×1600 分享图，汇总展示所有游戏（按内容分类）。

- **形态**：单文件 `index.html`（内联 CSS/JS），零依赖
- **数据本地**：游戏列表在页面内 `GAMES.categories` 数组中维护
- **移动适配**：触屏 + 微信 webview
- **部署域名**：`https://game-list-arv.pages.dev/`

## 本地运行

```sh
cd game-list
python3 -m http.server 8000
# 浏览器访问 http://localhost:8000
```

> 分享卡片二维码依赖 `api.qrserver.com`，需 http(s) 来源加载，**不要用 `file://` 直接打开**。
> 所有二维码加载完成后解锁「下载 PNG」按钮。

## 分类

按游戏**内容**划分（非按目录名）：

- **模拟体验**
- **休闲娱乐**
- **玄学趣玩**
- **脑力挑战**

## 添加 / 修改游戏

编辑 `index.html` 中 `GAMES.categories` 数组，每个游戏需提供 `name`、`desc`、`url`：

```js
{ name: '游戏名', desc: '简介', url: 'https://xxx.pages.dev/' }
```

- 跳过非游戏项目：`game-api`、`match-3`（素材阶段）、`game-list` 自身
- 已有 `pages.dev` 域名的项目填写完整 `https://` URL；未部署的留空
- 空项目（仅有素材，无 `index.html`）标记为「即将上线」

## 自动扫描更新

下次粘贴下面这段提示词，可自动从 `game` 目录扫描并更新列表（画布高度随游戏数量自适应）：

```
从 /Users/welkin/Code/game/ 扫描所有游戏项目，自动更新 index.html 的游戏列表，画布高度随游戏数量自适应。

规则：
1. 扫描每个子目录，读取 index.html 获取游戏名称（<title>）、简介、部署 URL（og:image 或 QR URL 中的 *.pages.dev）
2. 跳过非游戏项目：game-list 自身、game-api、match-3（仅素材）
3. 按游戏内容自动判断分类，分类名不限于现有四类，可根据游戏内容自动扩展新分类
4. 空项目（仅有 doc/ 或 .idea/，无 index.html）标记为「即将上线」，URL 留空
5. 已有 pages.dev 域名的项目填写完整 https:// URL；未有域名的填写留空
6. 更新 GAMES.categories 后直接浏览器打开 index.html 验证效果
```

## 约定

- 不初始化 git，不提交代码到 git（继承根 `AGENTS.md`）
- 无构建 / 测试 / 格式化流程，编辑后直接浏览器验证即可
- 样式：借鉴 Emil 设计（apple-design 风格）+ CSS 变量统一主题
