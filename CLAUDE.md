# edgetunnel — CLAUDE.md

## Project Overview

**edgetunnel** is a Cloudflare Workers/Pages application (version 2.0) that implements a VLESS/Trojan proxy tunnel. It runs as a single JavaScript Worker file and exposes an admin panel, subscription generation, and WebSocket-based proxy forwarding.

The entire application lives in one file: `_worker.js` (~1459 lines). There is no build step, no package manager, and no node_modules. Deployment is done by uploading the file to Cloudflare Workers or Pages.

## Repository Layout

```
_worker.js          — entire application (1459 lines)
wrangler.toml       — Cloudflare Workers config (name, compatibility_date)
.github/
  workflows/
    sync.yml        — daily cron that syncs forks from upstream cmliu/edgetunnel
README.md           — Chinese-language user-facing deployment guide
```

## Runtime Requirements

| Requirement | Detail |
|---|---|
| Platform | Cloudflare Workers or Cloudflare Pages (with Functions) |
| KV binding | Variable name **`KV`** — must be bound to a KV namespace |
| Admin password | One of: `ADMIN`, `admin`, `PASSWORD`, `password`, `pswd`, `TOKEN`, `KEY` env vars |
| CF socket API | Uses `cloudflare:sockets` (`connect`) for raw TCP |

## Environment Variables

| Variable | Required | Purpose |
|---|---|---|
| `ADMIN` / `PASSWORD` / `pswd` / `TOKEN` | Yes | Admin panel login password |
| `KEY` | No | Quick subscription shortcut (`/{KEY}` → `/sub`) and encryption salt |
| `UUID` | No | Force a fixed UUID; if omitted, UUID is derived from `MD5MD5(ADMIN + KEY)` |
| `PROXYIP` | No | Override the default Cloudflare egress IP(s). Comma-separated list; one is chosen randomly per request |
| `HOST` | No | Override the hostname used in node links / subscription content |
| `URL` | No | Homepage disguise: reverse-proxy this URL for non-authenticated visitors |
| `GO2SOCKS5` | No | Comma/newline list of domain glob patterns forced through SOCKS5 |
| `KV` | Yes | Cloudflare KV namespace binding |

## KV Storage Keys

| Key | Content |
|---|---|
| `config.json` | Main configuration object (see `读取config_JSON`) |
| `tg.json` | Telegram bot credentials (`BotToken`, `ChatID`) |
| `cf.json` | Cloudflare API credentials (`Email`, `GlobalAPIKey`, `AccountID`, `APIToken`) |
| `ADD.txt` | Custom preferred IP list (newline/comma separated) |
| `log.json` | Access log array (up to last N entries) |

## HTTP Route Map

```
GET  /login                   — login page (served from Pages static)
POST /login                   — form auth, sets auth cookie on success
GET  /logout                  — clears auth cookie, redirects to /login
GET  /admin                   — admin panel (cookie-gated)
GET  /admin/config.json       — return current config_JSON
POST /admin/config.json       — save config to KV
GET  /admin/cf.json           — return raw request.cf object
POST /admin/cf.json           — save Cloudflare API credentials
POST /admin/tg.json           — save Telegram bot config
GET  /admin/ADD.txt           — return custom IP list
POST /admin/ADD.txt           — save custom IP list
GET  /admin/log.json          — return request logs
GET  /admin/init              — reset config to defaults
GET  /admin/check?socks5=...  — test SOCKS5 / HTTP proxy reachability
GET  /admin/getADDAPI?url=... — validate an optimized IP API URL
GET  /admin/getCloudflareUsage — query CF usage via stored credentials
GET  /sub?token=...           — subscription endpoint (token-protected)
GET  /locations               — proxy to speed.cloudflare.com/locations
GET  /{KEY}                   — quick subscription redirect (if KEY is set)
WS   /*                       — VLESS/Trojan proxy tunnel
```

### Admin Authentication

Cookie: `auth = MD5MD5(User-Agent + KEY + ADMIN_PASSWORD)`  
Cookie max-age: 86400 s (1 day).

### Subscription Token

`token = MD5MD5(host + userID)` — validated on `/sub` requests.

## Proxy Tunnel Architecture

```
Client (WS upgrade)
  └→ 处理WS请求()           — accept WebSocket, pipe to WritableStream
       ├→ 解析木马请求()     — detect Trojan: SHA224(password) header + CRLF
       ├→ 解析魏烈思请求()   — detect VLESS: UUID + address type parsing
       ├→ forwardataTCP()    — open cloudflare:sockets TCP to target host
       │    ├→ socks5Connect() — optional SOCKS5 egress
       │    └→ httpConnect()   — optional HTTP proxy egress
       └→ forwardataudp()    — DNS queries over UDP (port 53 only)
```

Protocol detection happens on the first WebSocket chunk:
- **Trojan**: bytes at offset 56–57 are `0x0D 0x0A` (CRLF after SHA-224 password)
- **VLESS**: otherwise (UUID-based header)

Speed test hostnames are blocked (`isSpeedTestSite()`).

## Subscription Generation

Supported output formats detected from User-Agent or `?target=` param:

| Format | Trigger |
|---|---|
| `mixed` (base64, v2rayN) | Default; also when subconverter UA detected |
| `clash` / `clash.meta` | UA contains `clash`/`meta`/`mihomo`, or `?clash` |
| `singbox` | UA contains `singbox`/`sing-box`, or `?sb`/`?singbox` |
| `surge&ver=4` | UA contains `surge`, or `?surge` |

**Local generation** (`config.优选订阅生成.local = true`): Worker fetches Cloudflare CIDR list from GitHub (ASN-aware: China Mobile/Unicom/Telecom get different lists), picks random IPs, builds node links directly.

**Remote generation** (`local = false`): Delegates to an external subscription converter at `config.优选订阅生成.SUB`.

**Subscription conversion** (non-`mixed` targets): Proxies through `config.订阅转换配置.SUBAPI`.

### Per-node Proxy Override (Path Parameters)

Clients can inject proxy settings per-connection via the WebSocket path:

```
/?proxyip=1.2.3.4            — override proxy IP
/proxyip=1.2.3.4
/proxyip.example.com         — domain starting with proxyip.

/socks5=user:pass@host:port  — SOCKS5 egress
/socks5://user:pass@host:port — SOCKS5 + global
/socks5://base64creds@host:port
/s5=...  /gs5=...            — shorthand

/http=user:pass@host:port    — HTTP CONNECT proxy
/http://user:pass@host:port  — HTTP proxy + global
/ghttp=...
```

## Naming Conventions

The codebase uses **Chinese (CJK) identifiers** extensively for variables, function names, and object properties. This is intentional and must be preserved. Do not rename them to English.

Examples:
- `反代IP` — reverse proxy IP
- `启用SOCKS5反代` — SOCKS5 proxy enabled flag
- `处理WS请求()` — WebSocket handler
- `读取config_JSON()` — config loader
- `优选订阅生成` — optimized subscription generation

English identifiers are used for low-level protocol functions (`forwardataTCP`, `connectStreams`, `makeReadableStr`, etc.) and Cloudflare API wrappers.

## Key Functions Reference

| Function | Line | Purpose |
|---|---|---|
| `fetch` (default export) | 6 | Main request router |
| `处理WS请求` | 330 | WebSocket proxy handler |
| `解析木马请求` | 385 | Trojan protocol header parser |
| `解析魏烈思请求` | 444 | VLESS protocol header parser |
| `forwardataTCP` | 479 | TCP stream forwarding with retry |
| `forwardataudp` | 522 | UDP/DNS forwarding |
| `connectStreams` | 561 | Pipe remote socket → WebSocket |
| `makeReadableStr` | 588 | WebSocket → ReadableStream adapter |
| `socks5Connect` | 641 | SOCKS5 handshake and tunnel |
| `httpConnect` | 677 | HTTP CONNECT tunnel |
| `surge` | 710 | Surge config post-processor |
| `请求日志记录` | 729 | Log request to KV + Telegram |
| `sendMessage` | 762 | Telegram Bot API notification |
| `MD5MD5` | 802 | Double MD5 via SubtleCrypto |
| `随机路径` | 816 | Generate random WebSocket path |
| `随机替换通配符` | 823 | Replace `*` wildcard in hostname |
| `批量替换域名` | 834 | Bulk replace placeholder domains in subscription content |
| `读取config_JSON` | 843 | Load/initialize config from KV |
| `生成随机IP` | 959 | Fetch CF CIDR list and pick random IPs |
| `整理成数组` | 981 | Parse newline/comma list to string array |
| `请求优选API` | 989 | Fetch optimized IPs from multiple API URLs (parallel, 3 s timeout) |
| `反代参数获取` | 1091 | Extract proxy parameters from request URL path/query |
| `获取SOCKS5账号` | 1149 | Parse SOCKS5 address string |
| `getCloudflareUsage` | 1184 | Query CF GraphQL API for Workers/Pages usage |
| `sha224` | 1241 | Pure-JS SHA-224 (for Trojan password) |
| `解析地址端口` | 1274 | Split `host:port` handling IPv6 brackets |
| `SOCKS5可用性验证` | 1314 | Check SOCKS5/HTTP proxy reachability |
| `nginx` | 1339 | Return fake nginx HTML page |
| `html1101` | 1369 | Return Cloudflare Error 1101 HTML page |

## Configuration Object Shape (`config_JSON`)

```json
{
  "TIME": "<ISO timestamp>",
  "HOST": "<worker hostname>",
  "UUID": "<vless/trojan UUID>",
  "协议类型": "vless",
  "传输协议": "ws",
  "跳过证书验证": true,
  "启用0RTT": true,
  "TLS分片": null,
  "随机路径": false,
  "PATH": "/<derived from proxy settings>",
  "LINK": "<single node link>",
  "优选订阅生成": {
    "local": true,
    "本地IP库": { "随机IP": true, "随机数量": 16, "指定端口": -1 },
    "SUB": null,
    "SUBNAME": "edgetunnel",
    "SUBUpdateTime": 6,
    "TOKEN": "<MD5MD5(host+uuid)>"
  },
  "订阅转换配置": {
    "SUBAPI": "https://SUBAPI.cmliussss.net",
    "SUBCONFIG": "<ACL4SSR config URL>",
    "SUBEMOJI": false
  },
  "反代": {
    "PROXYIP": "auto",
    "SOCKS5": { "启用": null, "全局": false, "账号": "", "白名单": ["..."] }
  },
  "TG": { "启用": false, "BotToken": null, "ChatID": null },
  "CF": {
    "Email": null, "GlobalAPIKey": null, "AccountID": null, "APIToken": null,
    "Usage": { "success": false, "pages": 0, "workers": 0, "total": 0 }
  }
}
```

## Development & Deployment

### No Local Dev Server

This project has no local development environment. All testing must be done on Cloudflare.

### Deploying with Wrangler

```bash
# Install wrangler (once)
npm install -g wrangler

# Deploy to Cloudflare Workers
wrangler deploy

# Set secrets
wrangler secret put ADMIN
```

### Deploying via Cloudflare Dashboard (Pages)

1. Upload `main.zip` (entire repo) to CF Pages → "Upload assets"
2. Set environment variable `ADMIN` to your password
3. Bind a KV namespace to variable `KV`
4. Bind a custom domain

### Required Bindings (wrangler.toml or Dashboard)

```toml
[[kv_namespaces]]
binding = "KV"
id = "<your-kv-namespace-id>"
```

### Wrangler Config (`wrangler.toml`)

```toml
name = "v20251104"
main = "_worker.js"
compatibility_date = "2025-11-04"
keep_vars = true
```

## CI/CD

`.github/workflows/sync.yml` runs daily (00:00 UTC) to sync forks from the upstream repo `cmliu/edgetunnel`. It only runs when `github.event.repository.fork` is true. Manual trigger via `workflow_dispatch` is also available.

## Security Notes

- **Sensitive values** are masked before being returned in API responses (`掩码敏感信息()` keeps first 3 and last 2 chars)
- **Auth cookie** is scoped to `Path=/; HttpOnly` with 1-day expiry
- **UUID** is deterministically derived from `MD5MD5(ADMIN + KEY)` so it changes if credentials change
- **Speed test sites** are blocked at the protocol level in `isSpeedTestSite()`
- The admin panel is served from an external static site (`https://edt-pages.github.io`) — the Worker only handles API routes under `/admin/*`

## External Dependencies

All external dependencies are fetched at runtime (no npm packages):

| Service | URL | Purpose |
|---|---|---|
| Pages static | `https://edt-pages.github.io` | Admin panel UI, login page, error pages |
| CF CIDR lists | `https://raw.githubusercontent.com/cmliu/cmliu/main/CF-CIDR*.txt` | Random IP pool for subscriptions |
| Default SUBAPI | `https://SUBAPI.cmliussss.net` | Subscription conversion backend |
| Default SUBCONFIG | ACL4SSR GitHub raw | Clash rule config |

## Common Patterns

**Adding a new admin API route:**
1. Add a branch in the `admin` block of the main `fetch` handler (lines ~50–184)
2. Check `访问路径` (case-insensitive) or `区分大小写访问路径` (exact case)
3. Gate behind the existing cookie check — do not bypass it

**Adding a new config field:**
1. Add to `默认配置JSON` in `读取config_JSON()` (line ~846)
2. Any migration logic for existing KV configs goes in the same function after loading

**Modifying subscription output:**
1. `mixed` format: lines ~229–278 in the main `fetch` handler
2. Clash/singbox/surge: handled via the subscription conversion proxy at `订阅转换URL`
3. Surge post-processing: `surge()` function at line 710
