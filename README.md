# xhs-publish-skill

把 markdown 渲染成小红书 1080×1440 图卡的 Claude Code skill。基于 Playwright（headless Chromium）截图。

> Fork & adapted from [comeonzhj/Auto-Redbook-Skills](https://github.com/comeonzhj/Auto-Redbook-Skills)。删掉了自动发布部分（小红书对 AI 托管账号严打），加了封面 header（头像+昵称+日期）、单层平铺 default 主题、本地字体、表格样式。

## 效果

- **封面**：emoji + 标题 + 副标题，顶部 header 带头像/昵称/日期
- **正文卡**：按 `---` 分页或自动按高度分页，渲染 markdown（标题/加粗/列表/引用/代码块/表格/图片全支持）
- **8 套主题**：default（推荐，单层平铺）/ professional / botanical / sketch / playful-geometric / neo-brutalism / retro / terminal

## 安装

```bash
# 1. 克隆到 Claude Code 的 skills 目录
git clone https://github.com/SpringShi798/xhs-publish-skill.git ~/.claude/skills/xhs-publish

# 或者放在项目级 skills 目录
git clone https://github.com/SpringShi798/xhs-publish-skill.git <your-project>/.claude/skills/xhs-publish

# 2. 装依赖
pip install markdown pyyaml playwright
playwright install chromium
```

## 用法

### Claude Code 里直接调用

输入 `/xhs-publish` 或描述「把这篇 md 渲染成小红书图」，Claude 会读 SKILL.md 并执行。

### 命令行直接调用

```bash
python3 ~/.claude/skills/xhs-publish/scripts/render_xhs.py \
  <md文件路径> \
  -t default \
  -m separator \
  -o <输出目录>/
```

参数：
- `-t` 主题：`default` / `professional` / `botanical` / `sketch` / `playful-geometric` / `neo-brutalism` / `retro` / `terminal`
- `-m` 分页模式：
  - `separator`（默认）：按 md 里的 `---` 手动分页
  - `auto-split`：根据内容高度自动切分
  - `auto-fit`：自动缩放文字以填满固定尺寸
  - `dynamic`：根据内容动态调整图片高度
- `-o` 输出目录

## md 文件格式

顶部 YAML frontmatter（封面信息）+ 正文（用 `---` 分隔多张卡片）：

```markdown
---
emoji: "🚀"
title: "5个效率神器"
subtitle: "让工作效率翻倍"
author: "你的昵称"
xhs_id: "你的小红书号"
date: "2026-05-14"
avatar: "/path/to/avatar.jpg"   # 可选；空则用 CSS 占位符圆圈
avatar_text: "策"                # avatar 为空时显示的占位字
---

# 第一张卡的标题

正文……

---

# 第二张卡的标题

正文……
```

## 关于 default 主题

经过和 small-redbook 用户的反复调优，default 主题的设计是：

- **纯色单层平铺**（无外边框、无圆角、无阴影）—— 视觉最干净
- **本地字体**（PingFang SC / Source Han Sans）—— 不依赖 Google Fonts，国内可用
- **42px 正文**，line-height 1.7，标题 72px
- **封面** header 在顶部（头像+昵称+日期），主体居中（emoji 上 / 标题中 / 副标题下）
- **正文卡** 无右下角水印（默认关闭），如需可手动改 renderer

如果想要"卡片浮在渐变背景"的效果，切换到 `professional` 主题。

## 重要警告

⚠️ **本仓库不提供任何自动发布到小红书的功能**。原 Auto-Redbook-Skills 的 `publish_xhs.py` 已删除。小红书对 AI 托管账号严打，自动发布会导致主号被封——发布请用手机端手动操作。

## 协议

MIT
