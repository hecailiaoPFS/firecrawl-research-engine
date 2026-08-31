# Firecrawl Research Engine (DSH Skill)

> A technical research and verification skill for LLM agents. It searches first with Firecrawl, cross-checks facts across sources, and degrades gracefully to the built-in web search — so answers are accurate, traceable, and resistant to hallucination.

**中文说明见文末 / Chinese README at the bottom ↓**

---

## What this is

A [DSH (DeepSeek Harness)](https://github.com/deepseek-ai/deepseek-harness) skill that turns an LLM agent into a technical research and verification engine. When the user asks about concrete parameters, versions, code details, or recent releases, the skill:

1. **Searches first** with Firecrawl's Search API (not scrape-only) — returns Top-N results *with full page Markdown bodies*
2. **Optionally re-scrapes** the 1-2 most authoritative URLs for missing details
3. **Degrades gracefully** to the built-in `web_search` when Firecrawl is unavailable
4. **Falls back** to local knowledge with an explicit ⚠️ warning when the web has nothing
5. **Verifies and cites**: cross-checks hard facts across ≥2 sources, always inline-cites `[来源: 标题 + URL]`, and surfaces contradictions side by side

Its purpose is to **minimize hallucination on technical questions**, with cost control (≤10 Firecrawl calls per conversation) and bounded latency (15s request timeout).

The skill works with **any agent that can load a SKILL.md and call MCP tools** — DSH, Claude Code, Codex, Cursor, etc. See `references/INSTALLATION.md`.

## Why Firecrawl (not just plain search)

| | Built-in `web_search` | Firecrawl Search |
| :--- | :--- | :--- |
| Search + snippets | ✅ | ✅ |
| **Full page Markdown body** | ❌ | ✅ (auto-scraped Top-N results) |
| JS rendering / anti-bot | ❌ | ✅ |
| Free tier | ✅ | ✅ (rate-limited, free to start) |

Firecrawl's Search API returns `data.web[]` — each entry carries `title`, `url`, `description`, and the **complete page body** in `markdown` (a single document page can be tens of KB). That is the "more technical details" the skill is built around. Full API reference (verified against the live v2 API): `references/FIRECRAWL-API.md`.

## Field-tested pitfalls (Windows)

Two non-obvious issues were hit and verified during real deployment. Read this section before installing on Windows or configuring the MCP server for the first time. Full write-ups are in `docs/TROUBLESHOOTING.md`.

### 1. The Windows sandbox breaks TLS for in-shell HTTPS calls

DSH's Windows sandbox executes commands under a `WRITE_RESTRICTED` restricted token, which permits writes only inside the workspace and a private temporary directory. A TLS handshake requires writing to the user's certificate/key caches (`%APPDATA%\Microsoft\Crypto\RSA`, `%APPDATA%\Microsoft\SystemCertificates`); those writes are denied, so every HTTPS call fails with `SEC_E_NO_CREDENTIALS (0x8009030e)` — while plain TCP and DNS still work.

This is a kernel-level access check on the token, **not** an antivirus issue. Disabling Windows Security Center does not help. The reliable fix is to use the **Firecrawl MCP server**, which is spawned by the DSH host process outside the command sandbox and therefore performs TLS normally. In-sandbox direct API calls would otherwise need `danger-full-access` mode (one approval prompt per call).

### 2. The first MCP launch can exceed the 60s connection timeout

The first `npx -y firecrawl-mcp` downloads the package and roughly 40 dependencies (1-2 minutes), which exceeds the MCP client's default 60-second connection timeout. The client disconnects and the server process exits with it. This is a timing issue, not a protocol incompatibility.

Let the first attempt fail, then trigger an HMR reload (edit `cordis.patch.yml`) or restart DSH; the npx cache is warm afterwards and the connection completes in seconds. Do not delete the npx cache directory, or the timeout returns.

## Requirements

- Any agent that loads SKILL.md skills (DSH, Claude Code, Codex, Cursor, …)
- Optional but recommended: a Firecrawl API key (free tier at <https://firecrawl.dev>) — set as env var `FIRECRAWL_API_KEY`
- Optional but recommended on Windows: the Firecrawl MCP server (`firecrawl-mcp` on npm) — see the pitfalls above

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

**Option A — Firecrawl MCP server (recommended):**

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

The skill then sees `mcp__firecrawl__search` / `mcp__firecrawl__scrape` etc. (25 tools) and uses them with no per-call approval prompts. On the first launch, see the timeout note in the pitfalls section above.

**Option B — Direct API calls (no MCP):** see `references/FIRECRAWL-API.md` (curl / pwsh / node examples, verified response shape `data.web[]`).

**Option C — Nothing at all:** the skill still works — it degrades to the built-in `web_search` and labels answers with 🔎. The anti-hallucination behavior (citation, cross-check, contradiction surfacing) is independent of Firecrawl.

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
- **Cost control**: ≤10 Firecrawl calls and ≤4 search rounds per conversation; 15s request timeout
- **Resilience**: missing key / blocked network / sandbox restrictions never dead-end the flow
- **Context-aware truncation**: keep the 3-5 most relevant paragraphs instead of a fixed 128K threshold
- **Progressive disclosure**: SKILL.md stays a tight operational reference; deep details live in `references/`

## How this differs from Firecrawl provider plugins

Community plugins such as [`dsh-web-search-firecrawl`](https://github.com/elves-ai/dsh-web-search-firecrawl) replace the *search backend* (the `web_search` tool's underlying provider). This skill is a **workflow layer on top**: it defines the research methodology (query planning → search → re-scrape → degrade → verify → cite) and works with *any* Firecrawl surface — MCP tools, direct API, or a provider plugin. The two are complementary: install a provider plugin to upgrade the engine, this skill to upgrade the methodology.

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
│   └── TROUBLESHOOTING.md      ← Windows sandbox TLS, first-launch timeout, etc.
├── LICENSE                     ← MIT
└── .gitignore
```

## License

MIT © 2026 — see [LICENSE](LICENSE).

---

# 中文说明

## 这是什么

一个 **DSH(DeepSeek Harness)技能**,将 LLM 用作深度技术调研与验证引擎,也适用于任何能加载 SKILL.md 并调用 MCP 的 agent(Claude Code / Codex / Cursor)。

1. **搜索优先**——使用 Firecrawl Search API(而非仅抓取 URL),一次返回 Top-N 结果**及完整页面 Markdown 正文**
2. **可选精抓**——对最权威的 1-2 个 URL 再 scrape 补充细节
3. **优雅降级**——Firecrawl 不可用时自动改用内置 `web_search`
4. **保底兜底**——联网无结果时退回本地知识,并加 ⚠️ 明确警告
5. **验证与引用**——硬事实尽量由 ≥2 个独立来源交叉确认,回答强制内联引用 `[来源: 标题 + URL]`,矛盾并列展示

目标是**在技术问题上减少幻觉**,同时控制成本(单次对话 Firecrawl ≤10 次)和延迟(单请求 15s 超时)。

## 实战排障经验(Windows)

以下两个问题来自实际部署中踩过并验证的坑,在 Windows 上安装或首次配置 MCP 前建议先阅读。完整记录见 `docs/TROUBLESHOOTING.md`。

### 1. Windows 沙箱会导致沙箱内 HTTPS 调用 TLS 失败

DSH 的 Windows 沙箱以 `WRITE_RESTRICTED` 受限令牌执行命令,只允许在工作区和私有临时目录内写入。TLS 握手需要写入用户证书/密钥缓存(`%APPDATA%\Microsoft\Crypto\RSA`、`%APPDATA%\Microsoft\SystemCertificates`),这些写入被拒绝后,所有 HTTPS 调用都会报 `SEC_E_NO_CREDENTIALS (0x8009030e)`,而普通 TCP 和 DNS 仍然正常。

这是令牌上的内核级访问检查,**不是**杀毒软件问题,关闭 Windows 安全中心也无法解决。可靠的做法是使用 **Firecrawl MCP server**——它由 DSH 宿主进程启动,位于命令行沙箱之外,TLS 正常;沙箱内直接调用 API 则需 `danger-full-access` 模式(每次调用触发一次审批)。

### 2. MCP 首次启动可能超过 60 秒连接超时

首次执行 `npx -y firecrawl-mcp` 需下载包及约 40 个依赖(1-2 分钟),超过 MCP 客户端默认的 60 秒连接超时,导致客户端断开、服务进程退出。这是时序问题,不是协议不兼容。

首次失败后,通过修改 `cordis.patch.yml` 触发 HMR 重连,或重启 DSH 即可;此后 npx 缓存已热,数秒内即可连接。不要删除 npx 缓存目录,否则超时问题会复现。

## 与 Firecrawl provider 插件的区别

社区插件(如 [`dsh-web-search-firecrawl`](https://github.com/elves-ai/dsh-web-search-firecrawl))替换的是**搜索后端**;本技能是**方法论工作流层**(检索词规划 → 搜索 → 精抓 → 降级 → 验证 → 引用),与任何 Firecrawl 接入方式(MCP / 直接 API / provider 插件)兼容,可叠加使用。

## 安装

1. 将 `SKILL.md` 放入对应环境的技能目录(见上表或 `references/INSTALLATION.md`)
2. 配置 Firecrawl(三选一):
   - **A(推荐)**:Firecrawl MCP server,配置见 [`examples/cordis.patch.yml`](examples/cordis.patch.yml)
   - **B**:直接调用 v2 API(`references/FIRECRAWL-API.md`)
   - **C**:不配置——自动降级内置 `web_search`,反幻觉机制仍生效

## 常见问题

详见 [docs/TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md),重点覆盖:

- **Windows 沙箱 TLS 失败**(`SEC_E_NO_CREDENTIALS`):命令行沙箱的受限令牌无法访问证书存储 → 用 MCP 方案规避
- **首次 npx 下载超时**:首次启动约 1-2 分钟 > MCP 60s 连接超时 → 缓存热后秒连,或触发 HMR 重连
- **Firecrawl 无 Key 也能用**:keyless 模式限流可用,配置 Key 获得更高限额

## 协议

MIT © 2026 — 详见 [LICENSE](LICENSE)。
