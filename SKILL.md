---
name: xhs-publish
description: |
  小红书发布物渲染流水线：输入飞书文档链接或本地 markdown，输出 1080×1440 小红书卡片图（封面 + 正文卡片）到 xhs-output/。
  默认主题 default（纯色单层平铺），封面带头像+昵称+日期 header。
  支持中途配图：通读全文 → 给出每段配图建议（A 真实图/插画 调用 imagine | B 信息图 调用 infographic | 跳过）→ 用户拍板 → 插回 md → 统一渲染。
  触发词：「/xhs-publish」「渲染小红书」「出小红书图」「把这篇做成小红书图」「小红书发布物」「生成小红书卡片」。
---

# /xhs-publish · 小红书发布物渲染流水线

## 目标

把一篇 markdown（飞书文档 or 本地文件）变成可以直接发布的小红书图片：
- 封面（cover.png）：emoji + 标题 + 副标题 + 顶部 header（头像+昵称+日期）
- 正文卡片（card_1.png ~ card_N.png）：按 `---` 分页或按内容高度自动分页
- 全部 1080×1440 PNG，输出到 `xhs-output/<date>-<sanitized-title>/`

## 适用边界

- ✅ 用于个人账号发布物的「定稿 md → 图片」转换
- ❌ **不做自动发布**——发布请手机端手动操作（小红书对 AI 托管账号严打）
- ❌ 不替代写作工作流——这个 skill 接收的是已定稿的 md

---

## 输入约定

接受以下两类输入之一：

1. **飞书文档链接**（`https://*.feishu.cn/docx/...` 或 `wiki/...`）
   - 用 `lark-cli` 拉取为 md（需自行安装并授权）
2. **本地 md 路径**（绝对路径或相对路径）

md 顶部需要有 YAML frontmatter，建议字段：

```yaml
---
emoji: "🚀"                  # 封面 emoji
title: "你的标题"             # 封面主标题
subtitle: "你的副标题"         # 封面副标题
author: "你的昵称"            # 封面 header 昵称
xhs_id: "你的小红书号"         # 备用，目前不显示
date: "2026-05-14"           # 封面 header 日期
avatar: ""                   # 头像图片绝对路径（空则用 CSS 占位符圆圈）
avatar_text: "策"            # avatar 为空时圆圈里的字
---
```

如果 frontmatter 不全，主动询问用户补齐——尤其 emoji/title/subtitle 是封面必需。

默认走 `auto-split` 模式：按内容高度自动分页，带标题孤儿保护（H1/H2/H3 不会孤零零留在卡片末尾）。如果想精确控制分页，也可以手动放 `---` 用 `separator` 模式。

### Blockquote（`>` 引用块）的使用准则

把 `>` 当成"舞台聚光灯"，不是"段落分隔符"。这句话从段落里抽出来单独贴在朋友圈也成立——上聚光灯；只是承上启下的过渡话，就让它待在段落里。

**✅ 该用 `>` 的 4 类**：

1. **金句 / 观点高潮**：作者想让读者圈起来转发的那一句结论性表达
2. **名人 / 外部引用**：直接引用第三方原话（吴恩达、乔布斯、论文等）
3. **规则 / 规范定义**：结构化的策略表达、SOP、判定条件，需要视觉切出来
4. **假设对话 / 反问**：模拟某个角色的发问体，制造"对话感"

**❌ 不该用 `>` 的**：

- 普通叙述、口语化插话、故事开场白（加 `>` 会切碎阅读流）
- 单纯的强调（用粗体 `**` 或 H3 标题更合适）
- 列表项里的小标签（用 `**xxx**：` 即可）

拉取/读取 md 之后，按这个准则过一遍正文，**主动建议增删 `>`**，并给出每处的理由（哪一类 / 为什么）。

---

## 工作流（共 4 步）

### Step 1：拿到 md

- 飞书链接 → 用 `lark-cli` 解析 wiki/docx token，再调 `/open-apis/docx/v1/documents/{token}/raw_content` 拿正文，存为 `xhs-output/<date>-<sanitized-title>/source.md`，再补 frontmatter
- 本地 md → 直接读

读取后用 Read 工具看一遍全文，确认能解析 frontmatter + 正文分段。

### Step 2：配图判断（关键步骤）

通读全文，按段输出**配图建议表**：

| 段位置 | 当前内容主旨 | 建议配图类型 | 配图描述 |
|---|---|---|---|
| 封面 | xxx | A · imagine | 极简插画：xxx |
| 第 1 段 | xxx | 跳过 | 文字本身够 |
| 第 2 段 | xxx | B · infographic | 数据对比卡：xxx |

**判断原则（默认跳过，只有以下两类才配）**：

- **A 类（imagine）**：文章**直接描述**一个具体可视化对象——产品 UI 截图、人物动作场景、地点、特定物件。不是"概念"，是"画面"。
- **B 类（infographic）**：内容是数据对比、流程步骤、框架结构、参数表等**结构化信息**。

**❌ 默认跳过的场景（占绝大多数）**：
- 抽象观点 / 金句 / 隐喻 → 纯文字本身更有力
- 论述、叙事、反思段落 → 加图反而稀释信息密度
- 仅为"美化"或"破单调"配图 → 这是配图最常见的错误动机，硬克制

**配图比例参考**：一篇 12-15 张卡片的长文，配图通常 ≤2 张。如果你建议表里 A/B 类超过 3 个，先停下重审——大概率过度配图了。

把建议表给用户，等用户单段单段拍板。

### Step 3：执行配图（仅对用户批准的段）

- A 类 → 调用 `imagine` skill 或其他 AI 画图工具
- B 类 → 调用 `infographic` skill 或其他信息图工具

把每张图的路径插回 md 对应段落末尾（markdown 图片语法）：

```markdown
（段落原文……）

![](./images/section-3.png)
```

中间配图存放在 `xhs-output/<date>-<sanitized-title>/images/`，md 里用相对路径引用。

### Step 4：渲染最终图片

```bash
python3 <skill 安装路径>/scripts/render_xhs.py \
  <最终 md 路径> \
  -t default \
  -m auto-split \
  -o xhs-output/<date>-<sanitized-title>/
```

参数：
- `-t` 主题：`default`（推荐）/ `professional` / `botanical` / `sketch` / `playful-geometric` / `neo-brutalism` / `retro` / `terminal`
- `-m` 分页模式：
  - `auto-split`（默认）：根据内容高度自动切分，带标题孤儿保护（H1/H2/H3 不会孤零零留在卡片末尾）
  - `separator`：按 md 里的 `---` 手动分页（适合想精确控制分页位置时）
  - `auto-fit`：自动缩放文字以填满固定尺寸
  - `dynamic`：根据内容动态调整图片高度（不限定 1440）
- `-o` 输出目录

渲染产物：
- `cover.png`
- `card_1.png` ~ `card_N.png`
- `images/`（Step 3 的中间配图）

⚠️ **小红书单条笔记最多 18 张图**——如果总数超了，调小字号 / 减少分段 / 合并段落。

---

## 风险提示

1. **绝不自动发布**：原 fork 的 `publish_xhs.py` 已删除。小红书严打 AI 托管账号
2. **路径锚定**：输出目录建议固定在 `xhs-output/`，不要散落
3. **文章名清洗**：`<sanitized-title>` = 把 title 里的空格/标点/emoji 替换成 `-`，长度限 30 字符
4. **目录命名加日期前缀**：实际输出目录是 `<date>-<sanitized-title>`，`<date>` 取自 frontmatter 的 `date` 字段（格式 `YYYY-MM-DD`）。例如 `2026-05-14-my-article/`。这样 ls 排序就是时间顺序

---

## 依赖

- Python 3：`markdown`、`pyyaml`、`playwright`（需要 `playwright install chromium`）
- 系统字体：`PingFang SC` / `Source Han Sans CN`（macOS 自带；其他系统装思源黑体即可）
- Google Fonts `@import` 已删除——离线可用，不依赖网络
- `lark-cli`（仅飞书输入时需要，可选）

---

## 错误处理

- frontmatter 缺失 emoji/title → 让用户补，不要硬造
- 飞书链接拉取失败 → 让用户检查 lark-cli 授权，不要绕开
- imagine/infographic 调用失败 → 把那段标"配图失败"，让用户决定重试 or 跳过
- playwright 渲染卡死 → 单独跑测试 md 验证环境

不要擅自降级到"绕过问题"的路径。先告诉用户根因。
