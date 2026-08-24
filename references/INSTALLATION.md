# 跨平台安装指南

本技能的核心指令写在 `SKILL.md`(标准 Markdown + frontmatter),可安装到多种 agent 环境。推荐通过 **Firecrawl MCP** 提供搜索/抓取能力(MCP 是通用协议,各环境都支持)。

## 方式一:Firecrawl MCP(推荐,通用)

### 1. 获取 API Key

注册 <https://firecrawl.dev> 获取免费 Key,设置为环境变量:

```bash
# 一次性设置(持久化)
export FIRECRAWL_API_KEY=fc-xxxx        # macOS/Linux: 写入 ~/.bashrc 或 ~/.zshrc
setx FIRECRAWL_API_KEY "fc-xxxx"        # Windows
```

### 2. 各环境接入 MCP

MCP 配置大同小异,核心是 spawn `firecrawl-mcp`(npm 包)并把 `FIRECRAWL_API_KEY` 传入:

#### DeepSeek Harness (DSH)

编辑 profile 的 `cordis.patch.yml`(完整示例见 `examples/cordis.patch.yml`):

```yaml
- insert:
    - id: mcp-firecrawl
      name: '@deepseek-ai/dsh-mcp-client'
      config:
        serverName: firecrawl
        transport: stdio
        command: npx.cmd        # Windows; macOS/Linux 用 npx
        args: ['-y', 'firecrawl-mcp']
        toolCallTimeoutMs: 120000
        env:
          FIRECRAWL_API_KEY: !!js process.env.FIRECRAWL_API_KEY
```

然后重启 `dsh web`。技能自动识别 `mcp__firecrawl__search` 等 25 个工具。

#### Claude Code / Codex / Cursor(或其他 MCP 客户端)

在客户端的 MCP 配置中注册同一 server:

```json
{
  "mcpServers": {
    "firecrawl": {
      "command": "npx",
      "args": ["-y", "firecrawl-mcp"],
      "env": { "FIRECRAWL_API_KEY": "fc-xxxx" }
    }
  }
}
```

各客户端的配置文件位置:

| 客户端 | 配置文件 |
| :--- | :--- |
| Claude Code | `~/.claude.json`(mcpServers 字段)或 `claude mcp add` 命令 |
| Codex | `~/.codex/config.toml` 或 `codex mcp add` |
| Cursor | Settings → MCP Servers |
| 通用 | 支持 MCP 的 agent 均可用此 server |

### 3. 安装技能指令

| 环境 | 放置位置 |
| :--- | :--- |
| DSH | `~/.dsh/skills/firecrawl-research-engine.md`(用户级)或 `<project>/.dsh/skills/`(项目级) |
| Claude Code | `~/.claude/skills/firecrawl-research-engine/SKILL.md` |
| Codex | `~/.codex/skills/firecrawl-research-engine/SKILL.md` |
| Cursor | `.cursor/skills/` 或全局 skills 目录 |

各环境加载 SKILL.md 后,模型即可按技能指令执行完整调研工作流(检索词规划 → Firecrawl 搜索 → 精抓 → 降级 → 验证引用)。

## 方式二:直接调用 API(无 MCP)

不使用 MCP 时,模型可用任意 HTTP 能力调用 Firecrawl v2 API(见 `references/FIRECRAWL-API.md`):

- 需要能发起 HTTPS 请求(沙箱环境可能受限,见 `docs/TROUBLESHOOTING.md`)
- 需要 `FIRECRAWL_API_KEY` 环境变量
- 技能第 0 步会检测到无 MCP 工具但 Key 存在,走 A 路线的直接 API 变体

## 方式三:零配置兜底

无 Key、无 MCP、无网络——技能自动降级为内置 `web_search`(或任何可用的搜索工具),反幻觉机制(引用、交叉验证、矛盾并列)依然生效。这也是技能在**未完成任何配置**时就能直接使用的原因。

## 验证安装

配置完成后,向 agent 提问触发技能:

> "用 Firecrawl 查一下 Redis 7.4 持久化配置的细节"

预期行为:
1. 技能激活,规划检索词
2. 调用 Firecrawl Search(或降级 web_search)
3. 回答带内联来源 `[来源:标题 + URL]`,硬事实标注"多源一致 / 仅单一来源"

若 Firecrawl 调用失败,检查 `docs/TROUBLESHOOTING.md` 中的已知问题(首次下载超时、Windows 沙箱 TLS 等)。
