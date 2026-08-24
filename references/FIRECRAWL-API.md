# Firecrawl v2 API 参考(实测验证)

> 本文件基于真实调用验证的 Firecrawl v2 API 细节。官方文档:https://docs.firecrawl.dev

## 认证

所有请求携带 Bearer Token:

```
Authorization: Bearer <FIRECRAWL_API_KEY>
```

无 Key 时 Firecrawl 提供限流的 keyless 模式(搜索/抓取可用,速率受限)。配 Key 获得更高限额与完整工具集。

## Search API(主入口)

`POST https://api.firecrawl.dev/v2/search`

### 请求体

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| `query` | string | **必填**。检索词 |
| `limit` | number | 返回条数,默认 10;建议 3-5 控制成本 |
| `scrapeOptions.formats` | string[] | `["markdown"]` 返回正文;`["markdown","links"]` 附链接 |
| `scrapeOptions.onlyMainContent` | boolean | `true` 丢弃导航/广告/页脚 |
| `scrapeOptions.actions` | object[] | 动态渲染等待:`[{ "type": "wait", "milliseconds": 2000 }]`(v2 写法;旧版用 `waitFor: 2000`) |

### 响应结构(重点!与直觉不同)

```
{
  "success": true,
  "data": {
    "web": [                 ← 结果在 data.web[],不是 data[]
      {
        "url": "...",
        "title": "...",
        "description": "...",      // Firecrawl 摘要(Markdown)
        "position": 0,
        "markdown": "...",         // ★ 页面完整正文,直接用于回答
        "metadata": {              // dateModified / statusCode / wordCount ...
          "dateModified": "...",
          "statusCode": 200,
          "wordCount": 1234
        }
      }
    ]
  },
  "creditsUsed": 5
}
```

**常见坑**:结果在 `data.web[]`(不是 `data[]`);正文在 `.markdown`(不是 `.content`);摘要字段叫 `description`。

### 实测数据点

- 一次搜索 `limit: 3` 约消耗 **5 credits**(含抓取)
- 单条技术文档的 `markdown` 可达 **32K+ 字符**(如 Redis 官方文档)
- `metadata.dateModified` 可用来判断页面时效性

## Scrape API(精抓单页)

`POST https://api.firecrawl.dev/v2/scrape`

```json
{
  "url": "https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/",
  "formats": ["markdown"],
  "onlyMainContent": true,
  "actions": [{ "type": "wait", "milliseconds": 2000 }]
}
```

响应:顶层 `data.markdown`(正文)、`data.metadata`(含 `dateModified`、`statusCode` 等)。

## 调用示例

### curl

```bash
curl -s -X POST "https://api.firecrawl.dev/v2/search" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $FIRECRAWL_API_KEY" \
  -d '{
    "query": "Redis 7.4 persistence configuration",
    "limit": 3,
    "scrapeOptions": {
      "formats": ["markdown"],
      "onlyMainContent": true
    }
  }'
```

### PowerShell(pwsh,无 MCP 时)

> ⚠️ Windows 沙箱注意:受限令牌下 HTTPS 会报 `SEC_E_NO_CREDENTIALS`,需在 `danger-full-access` 模式运行,或直接用 MCP(推荐)。详见 `docs/TROUBLESHOOTING.md`。

```powershell
$key = $env:FIRECRAWL_API_KEY
$body = @{
  query = "Redis 7.4 persistence configuration"
  limit = 3
  scrapeOptions = @{ formats = @("markdown"); onlyMainContent = $true }
} | ConvertTo-Json -Depth 5
$raw = Invoke-WebRequest -Uri "https://api.firecrawl.dev/v2/search" `
  -Method Post -Headers @{ Authorization = "Bearer $key" } `
  -ContentType "application/json" -Body $body -TimeoutSec 20 -UseBasicParsing
$json = $raw.Content | ConvertFrom-Json
# 结果在 $json.data.web[],正文 .markdown,摘要 .description
$json.data.web | ForEach-Object { "$($_.title) | $($_.url) | markdown=$($_.markdown.Length)字" }
```

### JavaScript / Node

```js
const res = await fetch('https://api.firecrawl.dev/v2/search', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json', Authorization: `Bearer ${process.env.FIRECRAWL_API_KEY}` },
  body: JSON.stringify({ query: 'Redis 7.4 persistence', limit: 3,
    scrapeOptions: { formats: ['markdown'], onlyMainContent: true } })
});
const { data } = await res.json();
for (const item of data.web) console.log(item.title, item.url, item.markdown.length);
```

## 版本说明

- 本参考基于 Firecrawl **v2** API(2026 实测)。
- 旧 v1 端点(`/v1/search` 返回扁平 `data[]`)仍可用但推荐 v2。
- 参数若有变动,以官方文档 https://docs.firecrawl.dev 为准。
