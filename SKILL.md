---
name: content-collector
description: >
  个人内容收藏与知识管理系统。收藏、整理、检索、二创。
  Use when: (1) 用户分享链接/文字/截图并要求保存或收藏,
  (2) 用户说"收藏这个"/"存一下"/"记录下来"/"save this"/"bookmark"/"clip this",
  (3) 用户要求按关键词/标签搜索之前收藏的内容,
  (4) 用户要求基于收藏内容生成小红书/社交媒体文案（二创/改写/洗稿）,
  (5) 用户发送 URL 并要求提取/总结/摘要内容,
  (6) 用户提到"之前看过一个..."/"上次收藏的..."等回忆性检索,
  (7) 用户转发或粘贴了一段内容（文章片段/推文/视频链接）并暗示想留存。
  即使用户没有明确说"收藏"，只要涉及保存外部内容、整理知识、检索历史收藏、
  或基于已收藏内容进行二次创作，都应使用此技能。
  已支持来源：博客、X/Twitter、网页、B站视频。
  内容类型：文字、长视频（B站可自动转录）、短视频。
  NOT for: 纯个人笔记/备忘录（直接写文件）、长文写作（用 internal-comms/docx）、
  公众号发布（用 wemp-ops）、小红书发布（用 xiaohongshu-ops）。
---

# Content Collector — 个人内容收藏系统

收藏好内容 → 结构化整理 → 关键词检索 → 二次创作

## 数据位置

### 主存储（AI 检索用）
`<WORKSPACE>/collections/`

```
collections/
├── articles/       # 文章、博客、长文
├── tweets/         # X/Twitter 推文、短内容
├── videos/         # 视频内容（转录+笔记）
├── wechat/         # 微信公众号文章
├── ideas/          # 零散想法、灵感
├── index.md        # 全局索引（自动维护）
└── tags.md         # 标签索引（自动维护）
```

### Obsidian 同步（人工浏览用，可选）
`<YOUR_OBSIDIAN_VAULT>/收藏/`

```
收藏/
├── 文章/           # ← collections/articles
├── 视频/           # ← collections/videos
├── 推文/           # ← collections/tweets
├── 公众号/         # ← collections/wechat
└── 想法/           # ← collections/ideas
```

**每次收藏时必须同时写入两个位置。** Obsidian 版本的差异：
1. **文件名**：用中文标题（从 frontmatter title 取），不用 `YYYY-MM-DD-slug` 格式
2. **标签**：frontmatter 保留 `tags` 数组 + 正文第一行加 `#tag1 #tag2 ...` 格式（Obsidian 图谱和搜索用）
3. **aliases**：frontmatter 加 `aliases: [title]`，方便 Obsidian 双链搜索
4. **目录映射**：`articles→文章`、`videos→视频`、`tweets→推文`、`wechat→公众号`、`ideas→想法`

**Obsidian 写入模板**（伪代码）：
```
target_dir = <YOUR_OBSIDIAN_VAULT>/收藏/{中文目录}/
filename = sanitize(title).md   # 去掉 <>:"/\|?* 等非法字符，截断80字符
content = 原始 frontmatter（加 aliases） + "\n\n" + "#tag1 #tag2 ..." + "\n\n" + body
```

## 收藏工作流

### Step 0：去重检查（每次收藏必做）
收藏任何内容前，先执行去重：
1. 如果有 URL → `obsidian search query="<domain/path>" total`（CLI 可用时）或 `grep -rl "<url>" collections/`（CLI 不可用时）
   - ⚠️ 搜索时**去掉 `https://` 前缀**，只用域名+路径部分（Obsidian 会把 `https:` 解析为操作符报错）
   - 例：`obsidian search query="github.com/user/repo" total`
2. 返回 > 0 → 已收藏，回复「📌 已存在：<标题>（之前已收藏过）」，终止流程
3. 返回 0 → 继续下方流程

### Supadata API（优先方案）
对于大部分 URL，优先使用 Supadata API 解析：
- 脚本：`SUPADATA_API_KEY=<key> python3 scripts/supadata_fetch.py <command> <url>`
- 环境变量：`SUPADATA_API_KEY` 存放在 TOOLS.md

| 内容类型 | 命令 | 说明 |
|---------|------|------|
| 网页/博客 | `web <url>` | 返回 Markdown 正文，1 credit |
| 视频转录 | `transcript <url> --text [--lang zh]` | YouTube/TikTok/X/Instagram/Facebook，1-2 credits |
| 社交媒体元数据 | `metadata <url>` | 标题、作者、互动数据，1 credit |

Supadata 不可用时的降级方案：
- 网页 → `web_fetch`
- B站 → 本地 bilibili 脚本（见下方）
- 需登录/内网 → Chrome Relay

### URL 内容（文章/博客/网页）
1. **优先** `supadata_fetch.py web <url>` 抓取正文
2. **降级** `web_fetch` 抓取正文
3. 提取标题、作者、发布日期、正文摘要、关键词
4. **Schema.org 结构化数据提取**（见下方「Schema.org 提取规范」）：尝试提取页面 JSON-LD，按类型补充 frontmatter 字段。获取不到静默跳过
5. **URL 路由增强**（见下方「URL 路由表」）：匹配到特定域名时，执行额外提取步骤
6. 🔴 **提取有价值的插图**（**必做步骤**，见下方「插图保存规范」）——不得跳过，写文件前必须完成。browser 不可用时走降级流程（见规范）
7. **主题关键词提取**：见下方「主题关键词提取规范」，提取 5-7 个精细概念切面，写入 frontmatter.themes
8. 生成 `collections/articles/YYYY-MM-DD-slug.md`（含插图引用）
9. **HTML 快照保存**（仅重要文章）：对 P0/P1 级别的文章，额外保存一份原始 HTML 到 `collections/articles/YYYY-MM-DD-slug.html`，防止源页面删除后内容丢失。普通收藏不保存快照（避免磁盘膨胀）
10. **同步到 Obsidian**（优先 CLI，降级直接写文件）→ `收藏/文章/{标题}.md`（含插图复制）
11. **Daily Note 联动** → `obsidian daily:append content="- 📌 收藏了 [[{标题}]]（{source}）| {一句话摘要}"`

### 视频内容（YouTube/TikTok/X/Instagram/Facebook）
1. **元数据**: `supadata_fetch.py metadata <url>`
2. **转录**: `supadata_fetch.py transcript <url> --text --lang zh`
   - **YouTube 降级**：supadata credit 用完或失败时，先 navigate 到视频页面再运行 `python3 ~/.openclaw/workspace/scripts/bb_browser_bridge.py youtube/transcript`（需要浏览器在视频页面上）
3. **内容提取**：基于转录文本提取核心观点、金句、要点
4. **精彩片段提取**（≥10分钟视频）：见下方「精彩片段提取规范」
5. **主题关键词提取**：见下方「主题关键词提取规范」
6. 生成 `collections/videos/YYYY-MM-DD-slug.md`
7. **同步到 Obsidian**（优先 CLI，降级直接写文件）→ `收藏/视频/{标题}.md`
8. **Daily Note 联动** → `obsidian daily:append`

### 纯文本/截图
1. 截图用 `image` 工具提取文字
2. 整理成结构化格式，来源可选补充

### B站视频（本地流程，Supadata 不支持 B站）
1. **元数据**：`python3 scripts/bilibili_extract.py <bvid_or_url>` → 标题、作者、时长、标签、数据指标
   - **降级**：`python3 ~/.openclaw/workspace/scripts/bb_browser_bridge.py bilibili/video <bvid>` → 结构化 JSON（view/like/danmaku/reply/favorite/coin/share）
2. **评论区**：
   - 无登录（API）：3条热门
   - 已登录（浏览器 shadow DOM）：20+条，见 `references/bilibili-comments.md`
   - **降级**：`python3 ~/.openclaw/workspace/scripts/bb_browser_bridge.py bilibili/comments <bvid>`
3. **视频转录**：`bash scripts/video_transcribe.sh <bvid_or_url> [--model base]`
   - **原生字幕优先**：自动检测 B站 CC 字幕（ai-zh > zh-CN > zh-Hans），有则直接下载转纯文本，跳过 whisper（秒级完成 vs 10+ 分钟）
   - **无原生字幕**：自动降级到 whisper 转录（yt-dlp 下载音频 + faster-whisper）
   - 依赖：yt-dlp、ffmpeg、faster-whisper（uv）、opencc
   - 需浏览器已登录B站（yt-dlp 读取 cookie）
   - 模型：tiny/base(默认)/small/medium，base 约10-15分钟转录46分钟视频
   - 输出：`/tmp/video_audio/{platform}_{id}_subtitle.txt`（原生）或 `_transcript.json` + `.txt`（whisper）
   - 输出 JSON 含 `subtitle_source: "native_cc"` 或 `"whisper"`，收藏时写入 frontmatter
   - **两步模式**（长视频推荐，可分步重试）：
     - `bash scripts/video_transcribe.sh --step download <url>` → 只下载音频/字幕
     - `bash scripts/video_transcribe.sh --step transcribe <audio_file>` → 只转录
   - 注意：ASR 有识别错误，专有名词需人工校验
   - **降级方案**（转录失败时）：
     - yt-dlp cookie 过期 → 提示用户在 openclaw browser 重新登录 B站
     - faster-whisper 不可用 → 尝试 `whisper` CLI（`pip3 install openai-whisper`）
     - 全部失败 → 仅保存元数据+评论，在收藏文件中标注"转录未获取，待补充"，不阻塞收藏流程
   - **句子合并**（Whisper 转录后自动执行）：`video_transcribe.sh` 内置调用 `sentence_merger.py`，将碎片段合并为完整句子，提升可读性和后续 AI 处理质量。合并结果在 `_merged.json` / `_merged.txt`
4. **内容提取**：基于转录文本提取核心观点、金句、要点
5. **精彩片段提取**（≥10分钟视频）：见下方「精彩片段提取规范」，提取 3-5 个最值得看的时间段，写入 frontmatter.highlights 和正文
6. **主题关键词提取**（所有内容）：见下方「主题关键词提取规范」，提取 5-7 个精细概念切面，写入 frontmatter.themes
7. 生成 `collections/videos/YYYY-MM-DD-slug.md`
8. **同步到 Obsidian**（优先 CLI，降级直接写文件）→ `收藏/视频/{标题}.md`
9. **Daily Note 联动** → `obsidian daily:append`

### 小红书/抖音视频（本地转录）
1. **转录**：`bash scripts/video_transcribe.sh <url> [--model base]`
   - 自动检测平台：`xiaohongshu.com/explore/*`、`xhslink.com/*`、`douyin.com/video/*`、`v.douyin.com/*`
   - 下载策略：优先 `--cookies-from-browser chrome`，失败后尝试无 cookie
   - whisper 转录（小红书/抖音无原生字幕）
   - **两步模式**同 B站
2. **内容提取**：基于转录文本提取核心观点、金句、要点
3. **主题关键词提取**：见下方「主题关键词提取规范」
4. 生成 `collections/videos/YYYY-MM-DD-slug.md`（frontmatter 含 `subtitle_source: whisper`）
5. **同步到 Obsidian** → `收藏/视频/{标题}.md`
6. **Daily Note 联动** → `obsidian daily:append`
7. **降级**：下载失败 → 仅保存 URL + 手动描述，frontmatter 加 `incomplete: true`

## 精彩片段提取规范

**目标**：从长视频转录中自动筛选 3-5 个最值得看的精华时间段，帮助用户快速定位价值内容。

### 触发条件
- 视频时长 ≥ 10 分钟
- 有完整转录文本（whisper 或 native_cc）
- < 10 分钟的短视频跳过此步骤

### 提取方式

使用以下 prompt 提取精彩片段（遵循 `references/prompt-engineering-template.md` 的 XML 结构化范式）：

```xml
<task>
<role>You are an expert content curator selecting the most valuable moments from a video transcript.</role>
<languageRequirement>IMPORTANT: You MUST generate all content in Chinese.</languageRequirement>
<context>
Title: {video_title}
Speaker: {video_author}
Duration: {duration}
</context>
<goal>From this {duration} video, select the 3-5 most compelling moments worth watching.</goal>
<instructions>
  <item>Each highlight must be a contiguous segment of 45-90 seconds.</item>
  <item>Title must be specific and ≤15 characters. No generic titles like "重要观点".</item>
  <item>Quote must be an exact verbatim match from the transcript.</item>
  <item>Timestamps in [MM:SS-MM:SS] format.</item>
  <item>Insight explains in one sentence why this moment matters.</item>
  <item>Distribute highlights across the full video timeline — don't cluster in the opening.</item>
  <item>Focus on: contrarian insights, vivid stories, data-backed arguments, practical frameworks, memorable quotes.</item>
</instructions>
<qualityControl>
  <item>Are highlights distributed across the video (beginning, middle, end)?</item>
  <item>Does each highlight stand alone without needing context?</item>
  <item>Is the quote a verbatim match from the transcript?</item>
</qualityControl>
<outputFormat>Return strict JSON: [{"title":"string","start":"MM:SS","end":"MM:SS","quote":"exact transcript text","insight":"one sentence why this matters"}]. No markdown.</outputFormat>
<transcript><![CDATA[
{transcript_with_timestamps}
]]></transcript>
</task>
```

### 写入格式

**frontmatter**：
```yaml
highlights:
  - title: "片段标题"
    start: "12:30"
    end: "13:45"
    quote: "原文引用"
    insight: "为什么值得看"
```

**正文**（在「核心观点」之后、「要点摘录」之前）：
```markdown
## 精彩片段

> AI 从 {duration} 视频中筛选的 {N} 个最值得看的片段

**1. [{start}-{end}] {title}**
> {quote}

{insight}
```

### 降级
- 转录文本过短（< 500 字）→ 跳过
- AI 提取失败 → 跳过，不阻塞收藏流程
- 产出不满 3 个 → 有几个写几个

## 主题关键词提取规范

**目标**：为每篇收藏内容提取 5-7 个精细概念切面，辅助选题和二创时快速判断"这篇可以从哪些角度写文章"。

### 与 tags 的区别

| | tags | themes |
|---|---|---|
| 粒度 | 大类（AI、产品） | 具体概念（Prompt 工程范式、用户分层策略） |
| 用途 | 检索/分类 | 选题/二创灵感 |
| 数量 | 3-5 | 5-7 |

### 触发条件
- **所有内容类型**（文章、视频、推文、笔记），只要有足够文本（≥ 200 字）
- 可以在生成 summary + tags 的同一步 AI 调用中一起提取，不需要额外调用

### 提取方式

使用以下 prompt（或合并到内容提取步骤中）：

```xml
<task>
<role>You are an expert content analyst and semantic keyword extraction specialist.</role>
<languageRequirement>IMPORTANT: You MUST generate all keywords in Chinese. Technical terms keep English in parentheses.</languageRequirement>
<context>
Title: {title}
Source: {source}
Author: {author}
</context>
<goal>Extract 5-7 core conceptual themes that precisely capture the main topics discussed.</goal>
<instructions>
  <item>Each keyword/phrase must be 2-6 Chinese characters (or 1-3 English words for technical terms).</item>
  <item>Each keyword must capture a meaningfully different angle, stakeholder, problem, or method.</item>
  <item>Specificity over Generality: "用户分层策略" > "用户运营".</item>
  <item>Cover different facets: challenges, solutions, frameworks, stakeholders, outcomes.</item>
  <item>Avoid synonyms or adjective swaps of earlier keywords.</item>
</instructions>
<qualityControl>
  <item>Would each theme alone spark a specific article idea?</item>
  <item>Are any two themes essentially the same concept reworded?</item>
</qualityControl>
<outputFormat>Return a JSON array of strings: ["theme1","theme2",...]. No markdown.</outputFormat>
<content><![CDATA[
{content_text_first_3000_chars}
]]></content>
</task>
```

### 写入格式

**frontmatter**：
```yaml
themes: ["Prompt工程范式", "AI Agent架构", "用户意图识别", "工具调用策略", "上下文窗口优化"]
```

### 降级
- 内容太短（< 200 字）→ 跳过
- AI 提取失败 → 跳过，用 tags 替代
- 输出不是数组 → 跳过

## Schema.org 提取规范

**目标**：自动从网页结构化数据中获取更准确的元数据，减少人工补充。

### 提取方法
1. 如果已用 `browser` 打开页面，`evaluate` 执行：
   ```javascript
   JSON.parse(document.querySelector('script[type="application/ld+json"]')?.textContent || 'null')
   ```
2. 如果只用了 `web_fetch`，用正则从 HTML 中提取 `<script type="application/ld+json">` 内容
3. 页面可能有多个 JSON-LD 块，取 `@type` 最相关的一个（优先 Article > NewsArticle > BlogPosting > WebPage）

### 字段映射

| Schema.org 字段 | → frontmatter 字段 | 覆盖规则 |
|---|---|---|
| `headline` / `name` | `title` | 仅当现有 title 为空或明显是 URL slug 时覆盖 |
| `author.name` / `author[0].name` | `author` | 仅当现有 author 为空时填充 |
| `datePublished` | `date_published` | 优先使用（比 meta tag 更准确） |
| `description` | `summary` | 仅当现有 summary 为空时填充 |
| `aggregateRating.ratingValue` | `schema_data.rating` | 直接写入 |
| `aggregateRating.reviewCount` | `schema_data.reviewCount` | 直接写入 |
| `@type` | `schema_type` | 直接写入 |
| 其他有价值字段 | `schema_data.*` | 按需写入，总共不超过 10 个 key-value |

### 注意
- **静默降级**：提取失败（无 JSON-LD、解析报错、字段为空）→ 跳过，不阻塞收藏流程
- **不覆盖已有**：已有明确值的字段不被 Schema.org 数据覆盖（Schema.org 是补充，不是权威源）
- **schema_data 精简**：只保留有信息量的字段（rating/price/duration/keywords 等），忽略 @context、publisher logo URL 等噪音

## URL 路由表

按域名/URL 模式匹配差异化提取策略。**匹配到时执行额外步骤，未匹配走默认流程。**

| URL 模式 | 自动 category | 额外提取 | 特殊处理 |
|----------|--------------|---------|---------|
| 内网域名（见下方清单） | `articles` | 标题、作者（如有） | **必须走 Chrome Relay**，见下方「内网文章收藏流程」 |
| `arxiv.org/abs/*` | `articles` | abstract、authors 列表、PDF 链接、subjects/categories | 标签自动加 `论文`；从 abs 页提取，不下载 PDF |
| `github.com/*/*`（repo 首页） | `articles` | stars、primary language、description、license、最近 commit 日期 | 提取 README 前 500 字作为正文摘要；区分 repo 页 vs 文件页（文件页走默认流程） |
| `mp.weixin.qq.com/*` | `wechat` | 公众号名称（`#js_name` 或 meta `og:site_name`） | 优先 browser 提取（见「站点选择器」） |
| `youtube.com/watch*` | `videos` | 频道名、时长、观看数 | 优先 Supadata transcript；Schema.org 通常有 VideoObject |
| `xiaohongshu.com/explore/*` 或 `xhslink.com/*`（视频类） | `videos` | 作者、点赞数 | `video_transcribe.sh` 本地转录（whisper）；非视频笔记走默认流程 |
| `douyin.com/video/*` 或 `v.douyin.com/*` | `videos` | 作者、点赞数 | `video_transcribe.sh` 本地转录（whisper） |
| `x.com/*/status/*` | `tweets` | likes、retweets、replies、是否 thread | thread（连续 tweet）自动展开全部推文 |
| `news.ycombinator.com/item*` | `articles` | HN 得分、评论数、top 3 高赞评论 | **同时**抓取原文链接的内容（HN 页面本身只有讨论） |
| `substack.com/p/*` 或 `*.substack.com/p/*` | `articles` | 作者、发布日期、likes | Substack JSON-LD 通常完整 |
| `medium.com/*` 或 `*.medium.com/*` | `articles` | 作者、claps、reading time | 注意付费墙截断（见「站点选择器」） |

### 路由匹配逻辑
1. 按 URL 正则从上到下匹配，**首个命中**生效
2. 命中后执行「额外提取」列的步骤，**叠加**到默认流程上（不替代）
3. `自动 category` 列覆盖默认分类逻辑
4. 未命中任何规则 → 走默认 URL 内容流程，category 默认 `articles`

### 扩展方式
后续遇到新的高频站点，直接在表格中追加一行。无需改代码逻辑。

## 内网文章收藏流程

### 内网域名清单
```
*.alibaba-inc.com    # 阿里内网通用（含 ata.alibaba-inc.com / aone / done 等）
*.aliyun-inc.com     # 阿里云内网
*.alibaba.net        # 阿里内部
*.taobao.org         # 淘系内网
*.antfin.com         # 蚂蚁内网（含 yuque.antfin.com）
*.atatech.org        # ATA 技术博客（ata.atatech.org）
alidocs.dingtalk.com # 钉钉文档（阿里内部文档）
```

### 识别方式
1. clip 脚本自动检测：消息以 `[内网]` 开头
2. 手动发送：用户发内网 URL 时，agent 按域名匹配识别

### 收藏步骤
1. **不调用 web_fetch / supadata**（内网必失败，浪费时间和 credit）
2. **提示老板点 Chrome Relay**：
   > 📌 检测到内网链接，我需要通过你的浏览器读取内容。
   > 请确认当前 Tab 已打开这个页面，然后**点一下 Chrome Relay 按钮**（toolbar 上，badge 变 ON），我来提取。
3. **等老板确认**（老板说"好了"/"点了"/"OK" 等）
4. **通过 user 提取**：
   ```
   browser(action=snapshot, profile="user")
   ```
   获取页面正文内容
5. **正常走收藏流程**：提取标题/作者/正文 → 生成收藏文件 → 更新索引 → 同步 Obsidian
6. **提取完成后提醒老板**可以关掉 Relay（可选，不关也没事）

### 降级处理
- 老板不方便点 Relay → 仅保存 URL + 标题（从消息或老板口述获取），frontmatter 加 `incomplete: true`，正文加 `> ⚠️ 内网文章，正文待补充。下次在浏览器打开时可重新收藏。`
- user 提取正文 < 200 字 → 同上降级，可能是页面需要额外操作（如展开全文）

### 标签
内网文章自动加标签 `内网`、`阿里`。

## 站点选择器

当 `web_fetch` 提取结果残缺（正文 < 200 字）时，自动触发 `browser evaluate` + CSS 选择器精准提取。

| 站点 | 常见问题 | CSS 选择器 | 降级策略 |
|------|---------|-----------|---------|
| 内网域名（见清单） | 无外部访问权限 | 按页面结构判断 | **必须 Chrome Relay**，见「内网文章收藏流程」 |
| `mp.weixin.qq.com` | web_fetch 经常拿不到正文或格式混乱 | `#js_content` | Chrome Relay 打开原始页面提取 |
| `medium.com` | 付费墙截断，web_fetch 只拿到前几段 | `article section` | 仅保存可见部分，frontmatter 加 `incomplete: true`，标注"内容可能不完整（付费墙）" |
| `substack.com` | 部分文章需登录/付费 | `.post-content, .body` | web_fetch 通常可用；失败时同 medium 处理 |
| `36kr.com` | 反爬严重，web_fetch 可能返回空 | `.article-content` | browser 打开提取 |

### 触发条件
- `web_fetch` / `supadata` 返回的正文 **< 200 字**
- 或者正文明显是错误页（包含"请在微信客户端打开"/"Enable JavaScript" 等）
- 命中以上条件 → 自动尝试 browser 方案

### 降级原则
- 选择器提取失败 → **不阻塞收藏**
- 保存已有内容 + frontmatter 加 `incomplete: true`
- 收藏文件正文顶部加 `> ⚠️ 本文内容可能不完整，原始页面提取受限。`

## 插图保存规范

🔴 **收藏文章时必须执行插图提取步骤，不得跳过。** 这是收藏流程的强制环节，不是可选步骤。
写收藏文件之前，先完成插图筛选和下载。

### 判断标准（哪些图值得保存）
- ✅ 架构图、流程图、框架图、对比图、数据可视化
- ✅ 概念说明图、示意图、信息图
- ❌ 装饰性 banner、logo、头像、广告
- ❌ 纯文字截图（直接引用文字更好）

### 操作流程
1. **发现插图**：用 browser `evaluate` 提取页面 `<img>` 列表（src + alt + caption）
2. **筛选**：排除装饰性图片（logo、小于 200px、SVG 图标等）
3. **下载**：用 CDN 原始 URL（避免 Next.js 等图片优化层），`curl -sL` 保存到：
   - collections 路径：`collections/{category}/images/{slug}/01-描述.png`
   - slug 与收藏文件的文件名 slug 一致
4. **嵌入收藏文件**：在正文中用 `## 插图` 章节，每张图包含：
   - `![alt](images/{slug}/filename.png)`
   - 斜体说明文字（来自 caption 或自行总结）
5. **同步到 Obsidian**：
   - 复制图片到 `<YOUR_OBSIDIAN_VAULT>/收藏/{中文目录}/images/{slug}/`
   - Obsidian 版本使用相同的相对路径引用

### 命名规范
- 文件名：`{序号}-{简短英文描述}.png`（如 `01-architecture-overview.png`）
- 目录名：与收藏文件 slug 一致（如 `evals-for-agents`）

### 降级处理（browser 不可用时）
1. 如果是通过 `web_fetch` 抓取的页面 → 从 HTML 中正则提取 `<img>` 标签的 src
2. 如果 URL 是 CDN 图片链接 → 直接 `curl -sL` 下载
3. 如果完全无法获取图片列表（如纯 API 抓取、无 browser 无 HTML）→ 在 frontmatter 中标注 `images_pending: true`，收藏文件正文加 `> ⚠️ 插图待补充（收藏时无法访问页面图片）`
4. **绝不静默跳过**——要么提取图片，要么显式标注 pending

### 自检信号
每次收藏流程中，写文件前默问自己：**「图片提了没？」**
- 提了 → 继续
- 没提且能提 → 立刻补
- 没提且不能提 → 标注 `images_pending: true`

### 注意
- 图片保存为本地副本，不依赖外部 URL（防止链接失效）
- 单篇文章通常 3-8 张有价值的图，不要过度收集
- 筛选后 0 张有价值的图也是正常结果（如纯文字文章），此时不加 `images_pending`，但要在流程中明确记录「已筛选，无有价值插图」

## 关联项目（自动匹配）

每次收藏内容后，自动将内容与当前活跃项目关联：

1. 读取 `<WORKSPACE>/memory/topics/projects.md` 获取活跃项目列表
2. 将收藏内容的标题、摘要、标签与每个项目的关键词匹配
3. 匹配到的项目写入收藏文件的 YAML frontmatter：
   ```yaml
   related_projects: ["wemp-ops", "xiaohongshu-ops"]
   project_notes: "这个案例可以用于公众号选题：AI调度人力的真实案例"
   ```
4. 同时在 `index.md` 中标注关联项目，方便按项目筛选素材

### 项目关键词（自动从 projects.md 提取）
- **wemp-ops**：公众号、写作、文章、排版、内容运营
- **xiaohongshu-ops**：小红书、笔记、种草、配图、短内容
- **content-collector**：收藏、知识管理、素材库

### 使用场景
- 写公众号文章前：`搜索关联 wemp-ops 的收藏` → 快速找到素材
- 写小红书前：`搜索关联 xiaohongshu-ops 的收藏` → 找到适合拆解的内容
- 选题会议：按项目汇总最近收藏 → 发现选题方向

## 存储格式

文件命名：`YYYY-MM-DD-slug.md`

```yaml
---
title: "标题"
source: "来源平台"
url: "原始链接"
author: "作者"
date_published: "发布日期"
date_collected: "收藏日期"
tags: [tag1, tag2, tag3]
category: "articles|tweets|videos|wechat|ideas"
language: "zh|en"
summary: "一句话摘要"
# Schema.org 增强（可选，自动提取）
schema_type: "Article|VideoObject|Product|Recipe|SoftwareApplication|..."
schema_data: { rating: 4.5, reviewCount: 120 }  # 原始结构化数据摘要，≤10 key-value
incomplete: false  # 可选，true = 内容提取受限（付费墙/反爬），正文可能不完整
# 主题关键词（可选，AI 自动提取）
themes: ["主题1", "主题2", "主题3"]  # 5-7 个精细概念切面，辅助选题/二创
# 视频专属（可选）
duration: "时长"
platform: "bilibili|youtube"
bvid: "BV号"
stats: { views: 0, likes: 0, comments: 0 }
subtitle_source: "native_cc|whisper"  # 字幕来源
# 精彩片段（视频专属，AI 自动提取）
highlights:
  - title: "片段标题"
    start: "MM:SS"
    end: "MM:SS"
    quote: "原文引用"
    insight: "为什么值得看"
---
```

### 内容结构

- **内容概览**（条件触发，见下方规则）— Mermaid 图，一图看懂全文
- **核心观点** — 3-7个要点
- **精彩片段**（视频类，≥10分钟视频）— AI 筛选的 3-5 个最值得看的时间段（见下方「精彩片段提取规范」）
- **要点摘录** — 原文金句（blockquote）
- **热门评论精选**（视频类）— 含点赞数
- **评论区观点摘要**（视频类）— 总结争议点
- **我的笔记** — 用户个人批注，后续补充
- **原文摘要** — 200-500字概要

### 英文内容翻译规范

当 `language: en` 时，收藏过程中的翻译遵循以下规则：

**翻译风格**（默认 storytelling，不需要每次询问）：

| 风格 | 效果 | 适用 |
|------|------|------|
| `storytelling` | 叙事流畅，过渡自然（**默认**） | 博客、观点文、技术分享 |
| `technical` | 精准简洁，术语密集 | API 文档、技术规范 |
| `conversational` | 口语化，像朋友聊天 | Twitter thread、播客转录、访谈 |
| `formal` | 正式结构化 | 学术论文、白皮书 |

**触发条件**：老板说"精翻"→ 用更仔细的翻译；说"学术风格"→ formal；默认不问直接用 storytelling。

**术语表**：翻译时参照 `<WORKSPACE>/references/glossary-ai-zh.md` 统一术语。首次出现的术语用 `中文（English）` 格式。

**欧化检查**：翻译后扫一遍欧化中文问题（多余连接词/被动语态/名词堆砌），参照 `wemp-ops/references/style-guide.md` 的"欧化中文自检"。

### 内容概览图生成规则

**触发条件**：正文 > 1000 字的 articles / wechat / videos（短推文、零散想法不画）。

**输出格式**：Mermaid 代码块，直接嵌入收藏文件的 `## 内容概览` 章节。Obsidian 原生渲染，不需要额外插件。

**图表类型自动选择**（按文章内容匹配）：

| 文章特征 | 图表类型 | Mermaid 语法 | 示例 |
|---------|---------|-------------|------|
| 方法论/框架/模型（分层、组件） | 思维导图 | `mindmap` | "AI PM 的 3 个核心能力" |
| 流程/步骤/演进（先后顺序） | 流程图 | `graph TB` | "AI 编程三次进化" |
| 对比/选择（A vs B） | 对比图 | `graph TB` + 并行 subgraph | "传统 PM vs AI PM" |
| 多实体互动/依赖关系 | 关系图 | `graph LR` | "Agent 各模块数据流" |
| 时间线/里程碑 | 时间线 | `graph LR` 线性 | "2024 AI 大事记" |
| 混合/不确定 | 思维导图 | `mindmap`（万能兜底） | — |

**生成要求**：
1. **节点文字用文章原始术语**，不用"概念1"、"模块A"等占位符
2. **层级不超过 3 层**——图是辅助理解，不是完整复述
3. **节点数量 5-15 个**——太少没信息量，太多一眼看不完
4. **遵守 Mermaid 语法规范**——详见 `references/mermaid-syntax-rules.md`，特别注意：
   - `1. ` 触发列表解析错误 → 用 `①` 或 `(1)` 或去掉空格
   - subgraph 带空格 → 用 `subgraph id["显示名"]` 格式
   - 节点引用用 ID 不用显示文本
   - 不要在节点文本中使用 Emoji
5. **配色使用语义色**：
   - 核心概念：`fill:#d3f9d8,stroke:#2f9e44`（绿）
   - 问题/挑战：`fill:#ffe3e3,stroke:#c92a2a`（红）
   - 方法/工具：`fill:#e5dbff,stroke:#5f3dc4`（紫）
   - 输出/结果：`fill:#c5f6fa,stroke:#0c8599`（青）

**嵌入格式示例**：
```markdown
## 内容概览

\`\`\`mermaid
mindmap
  root((AI PM 核心能力))
    问题定义
      从模糊需求到精确问题
      判断什么值得做
    上下文质量
      Context Engineering
      给 Agent 正确的信息
    判断力
      评估 Agent 产出
      知道何时人工介入
\`\`\`
```

**不做什么**：
- 不生成独立图片文件——Mermaid 文本直接嵌入 Markdown
- 不画太复杂的图——收藏文件的图是"速览"，不是完整笔记

## Obsidian CLI 集成

### 概述
使用 Obsidian CLI（`obsidian` 命令）直接操作 KaiVault，替代手动写文件。CLI 通过 IPC 连接运行中的 Obsidian App，写入后自动触发索引更新。

### 可用性检测与降级
每次收藏操作开始时，先检测 CLI 是否可用：
```bash
obsidian vault 2>/dev/null
```
- 返回 vault 信息 → CLI 可用，使用 CLI 写入
- 返回错误或超时 → Obsidian 未运行，**降级到直接写文件**（原有方式，写入 `<YOUR_OBSIDIAN_VAULT>/收藏/` 目录）

降级对用户无感，不需要提示。CLI 可用时优先用 CLI。

### 收藏前去重
在执行收藏流程之前，先用 CLI 搜索是否已收藏过：
```bash
obsidian search query="<domain/path>" total
```
- ⚠️ **去掉 `https://` 前缀**，只搜域名+路径（`https:` 会被 Obsidian 解析为操作符报错）
- 同时去掉 query params（`?utm_*` 等）
- 返回 > 0 → 已收藏，告知用户「📌 已存在：<标题>（之前已收藏过）」，不重复收藏
- 返回 0 → 继续收藏流程

CLI 不可用时降级到 `grep -rl "<url>" collections/`。

### CLI 写入 Obsidian
收藏文件写入 `collections/` 后，用 CLI 同步到 Obsidian（替代直接写文件）：

```bash
# 创建笔记
obsidian create path="收藏/{中文目录}/{标题}.md" content="<完整内容>" overwrite

# 设置 frontmatter 属性
obsidian property:set path="收藏/{中文目录}/{标题}.md" name="aliases" value="[{title}]" type=list
obsidian property:set path="收藏/{中文目录}/{标题}.md" name="tags" value="[tag1,tag2]" type=list
obsidian property:set path="收藏/{中文目录}/{标题}.md" name="url" value="{url}" type=text
obsidian property:set path="收藏/{中文目录}/{标题}.md" name="source" value="{source}" type=text
obsidian property:set path="收藏/{中文目录}/{标题}.md" name="date_collected" value="{date}" type=date
```

**实际操作时**：优先用 `obsidian create` 一次性写入完整内容（含 frontmatter YAML 头），减少多次 CLI 调用。`property:set` 仅在需要**追加或修改**已有笔记属性时使用。

### Daily Note 联动
每次收藏成功后，追加一条记录到当天的 Daily Note：
```bash
obsidian daily:append content="- 📌 收藏了 [[{标题}]]（{source}）| {一句话摘要}"
```
CLI 不可用时跳过此步（不阻塞收藏流程）。

## Obsidian 同步规范

**优先使用 Obsidian CLI**（见上方「Obsidian CLI 集成」）。CLI 不可用时降级到以下直接写文件方式。

每次写入 `collections/` 后，**必须**同时写入 Obsidian 版本。步骤：

1. 从刚写入的收藏文件读取 frontmatter
2. 构建 Obsidian 版本：
   - 保留 frontmatter 的 `title, source, url, author, date_published, date_collected, category, language, summary, duration, platform, bvid, tags`
   - 添加 `aliases: [title]`
   - 正文第一行加 `#tag1 #tag2 ...`（标签中的空格替换为 `_`）
3. 文件名 = `sanitize(title).md`（去掉 `<>:"/\|?*`，截断 80 字符）
4. 写入到 `<YOUR_OBSIDIAN_VAULT>/收藏/{中文目录}/`

目录映射：
| collections 目录 | Obsidian 目录 |
|---|---|
| articles/ | 文章/ |
| videos/ | 视频/ |
| tweets/ | 推文/ |
| wechat/ | 公众号/ |
| ideas/ | 想法/ |

**注意**：Obsidian 版本是 collections 的只读镜像。编辑应在 collections 原文件上进行，然后重新同步。

## 索引

每次收藏后更新：
- **index.md** — 按月份倒序，含标签和来源
- **tags.md** — 按标签聚合所有收藏

## 检索

1. 标签匹配：`tags.md` 中查找
2. 全文搜索：`grep -ril "keyword" collections/`
3. 返回匹配列表 + 摘要

## 二创 — 小红书内容生成

1. 按选题/标签从收藏库筛选素材
2. 参考 `references/xiaohongshu-style.md` 写作风格
3. 生成：emoji标题 + 口语化正文(300-800字) + 话题标签(5-10个)
4. 发布配合 `xiaohongshu-ops` 技能

## 联动 — 公众号写作供料

收藏库是 `wemp-ops` 公众号写作的素材来源之一。wemp-ops 选题时会自动检索收藏库：
- 标签匹配：`tags.md` 按关键词查找相关收藏
- 全文搜索：`grep -ril` 搜索 collections/ 目录
- 收藏文件中的"核心观点"、"要点摘录"、"我的笔记"可直接用于文章引用

**收藏时的写作友好建议**：
- `summary` 写清楚，便于快速判断是否相关
- `tags` 覆盖主题关键词，提高检索命中率
- "我的笔记"多写自己的思考和联想，这些是文章的独特视角

## 分类规则

| 来源 | 目录 | 备注 |
|------|------|------|
| 博客/网页 | `articles/` | 默认归类 |
| X/Twitter | `tweets/` | 短内容/thread |
| 微信公众号 | `wechat/` | 中文长文 |
| B站/YouTube/抖音 | `videos/` | B站可自动转录 |
| 零散想法 | `ideas/` | 非外部来源 |

## 标签规范

- 中文为主，英文专有名词保持英文
- 每条 3-8 个标签
- 常用：`AI`、`产品设计`、`电商`、`运营`、`技术`、`商业`、`创业`、`效率工具`、`思维模型`、`管理`

## 写作素材交接（Skill Handoff）

收藏完成后，检查内容是否与选题池中的选题相关：

1. 读 `<WORKSPACE>/collections/topics/topic-pool.md`（公众号）和 `<WORKSPACE>/collections/topics/xhs-topic-pool.md`（小红书）
2. 如果收藏内容的标签/主题与某个选题匹配，追加到 `<WORKSPACE>/temp/handoffs/collector-to-writing.md`：
   ```markdown
   ### <收藏标题>
   - 关联选题：<匹配到的选题>
   - 来源：<URL>
   - 收藏时间：<日期>
   - 可用角度：<这个素材可以怎么用在文章里，1-2句话>
   ```
3. 追加模式（append），不覆盖已有内容
4. 如果没有匹配的选题，跳过此步（不创建空文件）

**注意**：此步骤是补充性的，不替代已有的"关联项目"功能。交接文件面向下游写作 skill，关联项目面向收藏库内部检索。

## 收藏结果通知

**每次收藏操作完成后（无论成功、重复、还是失败），必须返回收藏结果摘要。**

通知内容格式（作为最终回复直接输出）：
- 成功：`📌 已收藏：<标题>\n核心：<一句话摘要>\n标签：<3-5个标签>`
- 重复：`📌 已存在：<标题>（之前已收藏过）`
- 失败：`❌ 收藏失败：<URL>\n原因：<失败原因>`

**⚠️ 不要调用 `message` 工具发送通知。** 直接以纯文本返回收藏结果即可——Gateway 的 deliver 机制会自动将结果推送到用户最后活跃的渠道（webchat/钉钉等）。在 hook session 中调 `message(channel=openclaw-weixin)` 会因缺少 contextToken 而失败。

## 命令速查

| 用户说 | 动作 |
|--------|------|
| "收藏这个 [URL/文字]" | 抓取→整理→存储→更新索引 |
| "搜索 [关键词]" | 搜索收藏库 |
| "最近收藏了什么" | 读 index.md |
| "关于 [标签] 的收藏" | 从 tags.md 筛选 |
| "用 [选题] 写篇小红书" | 二创生成 |
| "给这篇加个笔记" | 更新"我的笔记"部分 |
| "删除这条收藏" | 移除并更新索引 + 删除 Obsidian 对应文件 |
| "重新同步到 Obsidian" | 全量重新同步：`python3 scripts/sync_to_obsidian.py`（skill 目录下） |

---

## 下一步建议（条件触发）

收藏完成后，根据内容特征判断是否推荐下一步。不是每次都推荐，只在有明确价值时才说。

| 触发条件 | 推荐 |
|---------|------|
| 收藏内容与老板公众号选题方向高度相关 | 「这篇素材适合二创公众号文章，需要的话用 wemp-ops 写。」 |
| 收藏内容适合小红书短图文（观点鲜明、有反差、可视化强） | 「这个素材适合做小红书笔记，需要的话用 xiaohongshu-ops 改写。」 |
| 收藏了某个 X 博主的多条内容（≥3 条） | 「这个博主收藏了好几条了，要不要用 x-profile-deep-dive 做个深度画像？」 |
| 收藏内容涉及技术方案/架构决策 | 「这个方案可以存到 memory 做长期参考，要记一下吗？」 |
