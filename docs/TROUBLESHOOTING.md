# Troubleshooting

Real issues hit and verified while developing this skill. If your setup fails, check here first.

## 1. Windows: all HTTPS calls fail with `SEC_E_NO_CREDENTIALS (0x8009030e)`

**Symptom**: In a sandboxed command shell (pwsh/bash under DSH's `workspace-write` or `read-only` mode), *every* HTTPS request fails — Firecrawl, GitHub, Baidu, anything. Plain TCP and DNS still work.

**Root cause**: DSH's Windows sandbox runs commands under a **`WRITE_RESTRICTED` restricted token** (ACL-based). The token only allows writes inside the workspace + a private temp dir. TLS handshakes need to *write* to the user's crypto/certificate caches:

```
%APPDATA%\Microsoft\Crypto\RSA\
%APPDATA%\Microsoft\SystemCertificates\
```

Those writes are denied → SChannel can't initialize credentials → `SEC_E_NO_CREDENTIALS`.

Note this is **not** a Defender/antivirus issue and **not** fixable by disabling Windows Security Center — it's a kernel access-check on the restricted token. Verified: same command succeeds under `danger-full-access` (full token) with Defender still running.

**Fixes (pick one)**:

- **Recommended: use the Firecrawl MCP server.** MCP servers are spawned by the DSH *host process*, outside the command sandbox — TLS works normally, and no per-call approval is needed. See `examples/cordis.patch.yml`.
- Or run the direct API call in `danger-full-access` mode (each call raises an approval prompt).
- Or accept automatic degradation to the built-in `web_search` (the skill handles it; answers are labeled 🔎).

## 2. MCP server process starts then exits / tools never appear

**Symptom**: You added the MCP plugin, restarted, but `mcp__firecrawl__*` tools never show up. Process tree shows `npx.cmd` chains appearing and dying.

**Root cause (most common)**: **first-launch download timeout**. The first `npx -y firecrawl-mcp` has to download the package + ~40 dependencies (1-2 minutes), but DSH's MCP client inherits the MCP SDK's default **60-second** connection timeout. The client gives up; the server process exits with it. Retries fail the same way until the reconnect budget is exhausted.

**Fix**:
1. Confirm the package downloaded (npx cache is warm): check `%LOCALAPPDATA%\npm-cache\_npx\<hash>\node_modules\firecrawl-mcp`.
2. Trigger a reconnect: edit `cordis.patch.yml` (any change — e.g. add `toolCallTimeoutMs: 120000`), which fires DSH's HMR reload, **or** restart DSH.
3. With a warm cache, connection succeeds in seconds. **Never delete the npx cache dir** or you're back to the timeout.

**Verify the server is healthy in isolation**:

```js
// node probe: spawn the server directly and speak MCP initialize
const { spawn } = require('child_process');
const child = spawn('npx.cmd', ['-y', 'firecrawl-mcp'], { stdio: ['pipe','pipe','pipe'] });
child.stdin.write(JSON.stringify({jsonrpc:'2.0', id:1, method:'initialize',
  params:{ protocolVersion:'2024-11-05', capabilities:{}, clientInfo:{name:'probe',version:'1.0'} }}) + '\n');
child.stdout.on('data', d => console.log('SERVER OK:', d.toString().slice(0, 120)));
```

If this works, the server is fine and the issue is on the client/connection side (see above).

## 3. `npx` / `npm` commands fail with "running scripts is disabled"

**Symptom**: `npx: File ...\npx.ps1 cannot be loaded because running scripts is disabled on this system`.

**Cause**: PowerShell execution policy blocks `.ps1` shims.

**Fix**: Use the `.cmd` variants explicitly — `npx.cmd`, `npm.cmd` — or set execution policy (`Set-ExecutionPolicy -Scope CurrentUser RemoteSigned`). The MCP config in `examples/cordis.patch.yml` already uses `npx.cmd`.

## 4. MCP tool calls time out

**Symptom**: `mcp__firecrawl__search` hangs or errors after ~60s.

**Fix**: Set `toolCallTimeoutMs` in the MCP plugin config (120000 recommended). Firecrawl search includes page scraping; slow sites can exceed the default 60s per-call timeout.

## 5. Firecrawl keyless vs keyed

Firecrawl works **without** a key in rate-limited "keyless" mode — useful for a quick test. For production, set `FIRECRAWL_API_KEY`. The skill auto-detects availability, so no config change is needed to switch.

## 6. "Not adapted to DSH" is usually not the issue

If the MCP handshake succeeds in isolation (see #2's probe) but not in DSH, it's almost always a timing/environment issue (timeout, sandbox, PATH), **not** a protocol incompatibility — DSH's MCP client is standard MCP via the official SDK. Check the npx cache timestamps vs. DSH host start time to confirm a first-launch timeout.
