---
name: firecrawl-research-engine
description: 用 Firecrawl 联网搜索获取指定主题的更多技术细节并交叉验证;Firecrawl 不可用时自动降级为内置 web_search,确保回答准确、可溯源、时效性强,最大限度避免幻觉。
whenToUse: 用户询问具体技术参数、版本号、代码实现、最新发布,或明确说"上网查一下 XXX 的细节"时
version: 2.2.0
metadata:
  author: firecrawl-research-engine
  license: MIT
  tags: [firecrawl, web-search, research, verification, anti-hallucination]
---

# Firecrawl 深度技术调研与验证引擎

## 一句话定位

**搜索优先(Firecrawl Search API)→ 精抓补充(scrape)→ 内置搜索降级 → 本地知识保底**的多级联网调研工作流,强制溯源与交叉验证,让技术回答准确、可查、不幻觉。

## 何时使用本技能(触发条件)

用户问题满足**任意一项**即激活:

1. **精确数据**:版本号、发布时间、性能指标(QPS、延迟)、价格、配置参数
2. **代码实现**:API 用法、SDK 示例、函数签名、配置项写法
3. **时效敏感**:"最新"、"近期"、"202X 年发布"的技术或政策
4. **权威要求**:官方推荐做法、安全规范、合规性要求
5. **超截止日期**:事件发生在模型训练数据截止日期之后
6. **显式请求**:用户说"查一下 / 搜索一下 / 用 Firecrawl 查 / 确认 XXX 的细节"

> 不满足上述条件的一般性问题,**不要**激活本技能,避免不必要的联网与成本。

## 核心工作流(严格按序执行)

### 第 0 步:接入检测
- **Firecrawl 可用**:会话出现 `firecrawl_search` / `firecrawl_scrape` / `mcp__firecrawl__*` 等工具,或 `FIRECRAWL_API_KEY` 非空 → **A 路线**
- **不可用**:无工具、无 Key → **B 路线**(内置 web_search)。**缺 Firecrawl 绝不放弃联网,直接降级**

### 第 1 步:检索词规划
- 把"XXX"拆成 **1-4 组精炼检索词**:中文原词 + 英文官方术语 + 限定词(`API reference`、`release notes`、`documentation`、`best practices`)
- 例:"Redis 7.4 持久化配置" → `Redis 7.4 持久化配置` / `Redis 7.4 persistence configuration` / `Redis 7.4 release notes persistence`
- 禁止:直接用整句口语提问当关键词

### 第 2 步(A 路线):Firecrawl Search(主梯队)
- 调用 **Search API**(`POST https://api.firecrawl.dev/v2/search`),关键参数:
  - `query`(检索词)、`limit: 3`(成本)、`scrapeOptions.formats: ["markdown"]`(要正文)、`scrapeOptions.onlyMainContent: true`(去噪音)
  - 动态渲染:`scrapeOptions.actions: [{ "type": "wait", "milliseconds": 2000 }]`(v2;旧版用 `waitFor: 2000`)
- 请求头:`Authorization: Bearer <FIRECRAWL_API_KEY>`
- **15 秒超时**:超时/404/空内容/反爬 → 立即进第 4 步,不阻塞用户
- **响应结构(实测)**:`data.web[]` 数组,每条含 `url`/`title`(来源)、`description`(摘要)、`markdown`(完整正文!)、`metadata`(含 `dateModified`/`statusCode` 可判时效)
- 成本:一次搜索约 5 credits,`limit` 越大越贵

### 第 3 步(A 路线补充):精抓高价值页面(可选)
- 从结果选**最权威 1-2 个 URL**(官方文档优先),用 **Scrape API**(`POST .../v2/scrape`)补细节(代码块、表格、参数值)
- 计入总预算,默认不做

### 第 4 步(B 路线 / 降级):内置 web_search
- 触发:Firecrawl 不可用或 A 路线全失败
- 用第 1 步的检索词调 `web_search`(queries 数组)
- 回答开头**前置标注**:"🔎 由于深度抓取工具未能获取该页面的完整内容,以下回答基于搜索引擎摘要,可能缺失部分细节,建议查阅原文。"
- 摘要与预训练知识冲突时,**以摘要为准**,提示"根据搜索结果,此处信息已更新"

### 第 5 步:联网无结果(第三梯队,保底)
- 触发:A、B 均无有效信息
- 退回本地知识,开头**强制警告**:"⚠️ 未能在网络上找到关于该问题的实时有效信息。以下回答仅基于模型内部知识,建议手动核实。"

### 第 6 步:内容清洗与截断
- 保留**正文、代码块、表格、带版本号的标题**;丢弃导航/广告/页脚
- **动态截断**:正文合计超剩余上下文一半,或单篇 >8-16K tokens → 只留与问题最相关的 3-5 个核心段落(含代码/表格),其余压成要点
- 多结果先去重(同 URL 只留一份)再截断

### 第 7 步:事实核对与合成
- 信息源优先级:**Firecrawl 正文 > 搜索摘要 > 本地知识**
- **强制内联引用**:Firecrawl → `[来源:页面标题 + URL]`;搜索 → `[来源:搜索引擎摘要 - 关键词:XXX]`
- **交叉验证**:版本号/日期/关键参数等硬事实,≥2 个独立来源确认时标注"多源一致";仅单一来源则写明"仅单一来源,建议复核"
- 矛盾时**并列展示**矛盾点并标注来源,不替用户选择

## 红线(不可违反)

| 场景 | 处理 |
| :--- | :--- |
| 多源矛盾 | 并列展示 + 各自来源,不代选 |
| 权威性不足 | 仅个人博客/非官方译文时,开头加"该信息来源于第三方,建议以官方原文为准" |
| 成本 | 单次对话 Firecrawl ≤10 次;web_search ≤4 轮;超限提示精简 |
| 超时 | 单次 Firecrawl >15s 立即降级 |
| 环境受限 | 沙箱禁网/TLS 受限 → 降级 web_search 或本地知识,不反复重试 |

## 部署与配置速查

- **Firecrawl MCP(推荐)**:见 `references/INSTALLATION.md` 与 `examples/cordis.patch.yml`;MCP 由宿主进程直连,可规避命令行沙箱 TLS 限制
- **直接 API**:见 `references/FIRECRAWL-API.md`(含 pwsh/curl 示例与响应解析)
- **无 Key 兜底**:keyless 限流可用;本技能无 Key 时自动走 web_search,反幻觉机制不变

## 参考文档

- `references/FIRECRAWL-API.md` — Firecrawl v2 API 完整参考(实测验证)
- `references/INSTALLATION.md` — 跨平台安装指南(DSH / Claude Code / Codex / Cursor)
- `docs/TROUBLESHOOTING.md` — 已知坑:Windows 沙箱 TLS、首次下载超时等
