# Firecrawl Research Engine (DSH Skill)

> 🔥 Deep technical research & verification skill for LLM agents: search first with Firecrawl, verify facts across sources, and gracefully degrade to built-in web search — so answers are accurate, traceable, and hallucination-resistant.

**中文说明见文末 / Chinese README at the bottom ↓**

---

## What this is

A [DSH (DeepSeek Harness)](https://github.com/deepseek-ai/deepseek-harness) skill that turns an LLM agent into a **deep technical research & verification engine**. When the user asks about concrete technical parameters, versions, code details, or recent releases, the skill:

1. **Searches first** with Firecrawl's Search API (not scrape-only) — gets Top-N results *with full page Markdown bodies*
2. **Optionally re-scrapes** the 1-2 most authoritative URLs for missing details
3. **Gracefully degrades** to the built-in `web_search` when Firecrawl is unavailable
4. **Falls back** to local knowledge with an explicit ⚠️ warning when the web has nothing
5. **Verifies & cites**: cross-checks hard facts across ≥2 sources, always inline-cites `[来源: 标题 + URL]`, and surfaces contradictions side by side

The whole point: **eliminate hallucination on technical questions**, with cost control (≤10 Firecrawl calls per conversation) and bounded latency (15s per request timeout).

The skill works with **any agent that can load a SKILL.md and call MCP tools** — DSH, Claude Code, Codex, Cursor, etc. (see `references/INSTALLATION.md`).

## Why Firecrawl (not just plain search)

| | Built-in `web_search` | Firecrawl Search |
| :--- | :--- | :--- |
| Search + snippets | ✅ | ✅ |
| **Full page Markdown body** | ❌ | ✅ (auto-scraped Top-N results) |
| JS rendering / anti-bot | ❌ | ✅ |
| Free tier | ✅ | ✅ (rate-limited, free to start) |

Firecrawl's Search API returns `data.web[]` — each entry carries `title`, `url`, `description`, and the **complete page body** in `markdown` (a single doc page can be tens of KB). That's the "more technical details" the skill is built around. Full API reference (verified against the live v2 API): `references/FIRECRAWL-API.md`.

## Requirements

- Any agent that loads SKILL.md skills (DSH, Claude Code, Codex, Cursor, …)
- Optional but recommended: a Firecrawl API key (free tier available at <https://firecrawl.dev>) — set it as env var `FIRECRAWL_API_KEY`
- Optional but best: Firecrawl MCP server (`firecrawl-mcp` on npm) — **recommended on Windows**, see Troubleshooting

## Installation

### 1. Install the skill instruction

| Environment | Location |
| :--- | :--- |
| DSH (user-level) | `~/.dsh/skills/firecrawl-research-engine.md` (Windows: `%USERPROFILE%\.dsh\skills\`) |
| DSH (project-level) | `<project>/.dsh/skills/` (takes precedence) |
| Claude Code | `~/.claude/skills/firecrawl-research-engine/SKILL.md` |
| Codex | `~/.codex/skills/firecrawl-research-engine/SKILL.md` |
| Cursor | `.cursor/skills/` or global skills dir |

Full per-environment instructions: `references/INSTALLATION.md`.

### 2. Configure Firecrawl (choose one)

**Option A — Firecrawl MCP server (recommended, universal):**

```bash
# any MCP-capable client
export FIRECRAWL_API_KEY=fc-xxxx
# register the server: npx -y firecrawl-mcp
```

DSH profile `cordis.patch.yml` example — full file in [`examples/cordis.patch.yml`](examples/cordis.patch.yml):

```yaml
- insert:
    - id: mcp-firecrawl
      name: '@deepseek-ai/dsh-mcp-client'
      config:
        serverName: firecrawl
        transport: stdio
        command: npx.cmd        # Windows; use `npx` on macOS/Linux
        args: ['-y', 'firecrawl-mcp']
        toolCallTimeoutMs: 120000
        env:
          FIRECRAWL_API_KEY: !!js process.env.FIRECRAWL_API_KEY
```

The skill then sees `mcp__firecrawl__search` / `mcp__firecrawl__scrape` etc. (25 tools) and uses them with **no per-call approval prompts**.

> ⚠️ First launch downloads `firecrawl-mcp` via npx (~1-2 min) which can exceed the MCP client's 60s connection timeout. Let it fail once, then trigger a reload (edit `cordis.patch.yml`) or restart — the npx cache is warm afterwards and connection succeeds in seconds.

**Option B — Direct API calls (no MCP):** see `references/FIRECRAWL-API.md` (curl / pwsh / node examples, verified response shape `data.web[]`).

**Option C — Nothing at all:** the skill still works — it degrades to the built-in `web_search` and labels answers with 🔎. Anti-hallucination behavior (citation, cross-check, contradiction surfacing) is independent of Firecrawl.

## How it works (the tier ladder)

```
User question (technical details?)
  │  trigger conditions match
  ▼
[0] Capability probe → Firecrawl available?
  ├─ YES ────────────────────────────────┐
  │  1. Query planning (1-4 keywords)     │
  │  2. Firecrawl Search (Top-N + body)   │
  │  3. Re-scrape top 1-2 pages (optional)│
  └──► 6. Clean & truncate (dynamic)      │
      7. Verify & synthesize w/ citations ─┘
  │
  ├─ NO / failure ──► [4] built-in web_search (🔎 labeled) ──► 6, 7
  │                       │ no results
  │                       ▼
  └─────────────────► [5] local knowledge (⚠️ warned) ──► answer
```

## Trigger conditions (any one activates)

1. Precise data: version numbers, release dates, performance metrics, prices, config params
2. Code specifics: API usage, SDK examples, function signatures, config syntax
3. Time-sensitive: "latest", "recent", "in 202X"
4. Authority-sensitive: official recommendations, security/compliance rules
5. Post-cutoff facts: events after the model's training data cutoff
6. Explicit request: "look up / search / verify XXX"

## Design principles

- **Traceability**: every answer inline-cites its source; degraded tiers are visibly labeled
- **Verification**: hard facts cross-checked across ≥2 independent sources when possible
- **Cost control**: ≤10 Firecrawl calls & ≤4 search rounds per conversation; 15s request timeout
- **Resilience**: missing key / blocked network / sandbox restrictions never dead-end the flow
- **Context-aware truncation**: keep the 3-5 most relevant paragraphs instead of a fixed 128K threshold
- **Progressive disclosure**: SKILL.md stays a tight operational reference; deep details live in `references/`

## How this differs from Firecrawl provider plugins

Community plugins like [`dsh-web-search-firecrawl`](https://github.com/elves-ai/dsh-web-search-firecrawl) replace the *search backend* (the `web_search` tool's underlying provider). This skill is a **workflow layer on top**: it defines the research methodology (query planning → search → re-scrape → degrade → verify → cite) and works with *any* Firecrawl surface — MCP tools, direct API, or a provider plugin. The two are complementary: install a provider plugin to upgrade the engine, this skill to upgrade the methodology.

## Repository layout

```
firecrawl-research-engine/
├── SKILL.md                    ← the skill itself (install this)
├── references/
│   ├── FIRECRAWL-API.md        ← Firecrawl v2 API reference (verified)
│   └── INSTALLATION.md         ← cross-platform install guide (DSH/Claude Code/Codex/Cursor)
├── examples/
│   └── cordis.patch.yml        ← DSH MCP configuration example
├── docs/
│   └── TROUBLESHOOTING.md      ← Windows sandbox TLS issue, first-launch timeout, etc.
├── LICENSE                     ← MIT
└── .gitignore
```

## License

MIT © 2026 — see [LICENSE](LICENSE).

---

# 中文说明

## 这是什么

一个 **DSH(DeepSeek Harness)技能**,把 LLM 变成**深度技术调研与验证引擎**,也适用于任何能加载 SKILL.md + 调用 MCP 的 agent(Claude Code / Codex / Cursor)。

1. **搜索优先**——用 Firecrawl Search API(不是只抓 URL),一次拿到 Top-N 结果**及完整页面 Markdown 正文**
2. **可选精抓**——对最权威的 1-2 个 URL 再 scrape 补细节
3. **优雅降级**——Firecrawl 不可用时自动改用内置 `web_search`
4. **保底兜底**——联网无结果时退回本地知识,并加 ⚠️ 明确警告
5. **验证与引用**——硬事实尽量 ≥2 个独立来源交叉确认,回答强制内联引用 `[来源: 标题 + URL]`,矛盾并列展示

目标:**在技术问题上杜绝幻觉**,同时控制成本(单次对话 Firecrawl ≤10 次)和延迟(单请求 15s 超时)。

## 与 Firecrawl provider 插件的区别

社区插件(如 [`dsh-web-search-firecrawl`](https://github.com/elves-ai/dsh-web-search-firecrawl))替换的是**搜索后端**;本技能是**方法论工作流层**(检索词规划 → 搜索 → 精抓 → 降级 → 验证 → 引用),与任何 Firecrawl 接入方式(MCP / 直接 API / provider 插件)兼容,可叠加使用。

## 安装

1. 把 `SKILL.md` 放入对应环境的技能目录(见上表或 `references/INSTALLATION.md`)
2. 配置 Firecrawl(三选一):
   - **A(推荐)**:Firecrawl MCP server,配置见 [`examples/cordis.patch.yml`](examples/cordis.patch.yml)
   - **B**:直接调用 v2 API(`references/FIRECRAWL-API.md`)
   - **C**:什么都不配——自动降级内置 `web_search`,反幻觉机制仍生效

## 常见问题

详见 [docs/TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md),重点覆盖:

- **Windows 沙箱 TLS 失败**(`SEC_E_NO_CREDENTIALS`):命令行沙箱的受限令牌无法访问证书存储 → 用 MCP 方案规避
- **首次 npx 下载超时**:首次启动约 1-2 分钟 > MCP 60s 连接超时 → 缓存热后秒连,或触发 HMR 重连
- **Firecrawl 无 Key 也能用**:keyless 模式限流可试,配 Key 获得更高限额

## 协议

MIT © 2026 — 详见 [LICENSE](LICENSE)。
