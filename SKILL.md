---
name: content-collector
description: >
  个人内容收藏与知识管理系统。收藏、整理、检索、二创。
  Use when: (1) 用户分享链接/文字/截图并要求保存或收藏,
  (2) 用户说"收藏这个"/"存一下"/"记录下来"/"save this",
  (3) 用户要求按关键词/标签搜索之前收藏的内容,
  (4) 用户要求基于收藏内容生成小红书/社交媒体文案,
  (5) 用户发送 URL 并要求提取/总结内容。
  已支持来源：博客、X/Twitter、网页、B站视频。
  计划支持：微信公众号、视频号、抖音。
  内容类型：文字、长视频（B站可自动转录）、短视频。
---

# Content Collector — 个人内容收藏系统

收藏好内容 → 结构化整理 → 关键词检索 → 二次创作

## 数据位置

`~/.openclaw/workspace/collections/`

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

## 收藏工作流

### URL 内容（文章/博客/网页）
1. `web_fetch` 抓取正文
2. 提取标题、作者、发布日期、正文摘要、关键词
3. 生成 `collections/articles/YYYY-MM-DD-slug.md`

### 纯文本/截图
1. 截图用 `image` 工具提取文字
2. 整理成结构化格式，来源可选补充

### B站视频（完整自动化流程）
1. **元数据**：`python3 scripts/bilibili_extract.py <bvid_or_url>` → 标题、作者、时长、标签、数据指标
2. **评论区**：
   - 无登录（API）：3条热门
   - 已登录（浏览器 shadow DOM）：20+条，见 `references/bilibili-comments.md`
3. **视频转录**：`bash scripts/bilibili_transcribe.sh <bvid_or_url> [model]`
   - 依赖：yt-dlp、faster-whisper（uv）、opencc
   - 需浏览器已登录B站（yt-dlp 读取 cookie）
   - 模型：tiny/base(默认)/small/medium，base 约10-15分钟转录46分钟视频
   - 输出：`/tmp/bilibili_audio/<BVID>_transcript.json` + `.txt`
   - 注意：ASR 有识别错误，专有名词需人工校验
4. **内容提取**：基于转录文本提取核心观点、金句、要点
5. 生成 `collections/videos/YYYY-MM-DD-slug.md`

### 其他视频平台（YouTube/抖音/视频号）
- 暂不支持自动处理
- 用户可提供转录文本或关键信息，手动整理

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
# 视频专属（可选）
duration: "时长"
platform: "bilibili|youtube"
bvid: "BV号"
stats: { views: 0, likes: 0, comments: 0 }
---
```

### 内容结构

- **核心观点** — 3-7个要点
- **要点摘录** — 原文金句（blockquote）
- **热门评论精选**（视频类）— 含点赞数
- **评论区观点摘要**（视频类）— 总结争议点
- **我的笔记** — 用户个人批注，后续补充
- **原文摘要** — 200-500字概要

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

## 命令速查

| 用户说 | 动作 |
|--------|------|
| "收藏这个 [URL/文字]" | 抓取→整理→存储→更新索引 |
| "搜索 [关键词]" | 搜索收藏库 |
| "最近收藏了什么" | 读 index.md |
| "关于 [标签] 的收藏" | 从 tags.md 筛选 |
| "用 [选题] 写篇小红书" | 二创生成 |
| "给这篇加个笔记" | 更新"我的笔记"部分 |
| "删除这条收藏" | 移除并更新索引 |
