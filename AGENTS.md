## 项目说明

游戏合集分享卡片生成器 — 单文件静态 HTML，Canvas 绘制 1200×1600 分享图片，汇总展示所有 AI 工具生成的游戏。

## 使用方法

- 直接浏览器打开 `index.html`，无需构建/安装/服务端
- 依赖 `api.qrserver.com` 生成二维码，需联网
- 所有二维码加载完成后解锁「下载 PNG」按钮

## 添加/修改游戏

编辑 `index.html` 中 `GAMES.categories` 数组，每个游戏需提供 `name`、`desc`、`url`：

```js
{ name: '游戏名', desc: '简介', url: 'https://xxx.pages.dev/' }
```

分类按游戏内容划分（非按开发目录），当前四类：模拟体验、休闲娱乐、玄学趣玩、脑力挑战。

## 约定

- 不初始化 git，不提交代码到 git（继承 `../AGENTS.md`）
- 无构建/测试/格式化流程，编辑后直接浏览器验证即可

## 自动扫描更新

下次粘贴下面这段话，自动从 game 目录扫描游戏并更新列表：

```
从 /Users/welkin/Code/game/ 扫描所有游戏项目，自动更新 index.html 的游戏列表，画布高度随游戏数量自适应。

规则：
1. 扫描每个子目录，读取 index.html 获取游戏名称（<title>）、简介、部署 URL（og:image 或 QR URL 中的 *.pages.dev）
2. 跳过非游戏项目：game-list 自身
3. 按游戏内容自动判断分类，分类名不限于现有三类，可根据游戏内容自动扩展新分类
4. 空项目（仅有 .idea/，无 index.html）标记为「即将上线」，URL 留空
5. 已有 pages.dev 域名的项目填写完整 https:// URL；未有域名的填写留空
6. 更新 GAMES.categories 后直接浏览器打开 index.html 验证效果
```

