# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a Railway deployment wrapper for **Openclaw** (an AI coding assistant platform). It provides:

- A web-based setup wizard at `/setup` (protected by `SETUP_PASSWORD` when set)
- Automatic reverse proxy from public URL → internal Openclaw gateway (requires `SETUP_PASSWORD` when set)
- Persistent state via Railway Volume at `/data`
- One-click backup export of configuration and workspace

The wrapper manages the Openclaw lifecycle: onboarding → gateway startup → traffic proxying.

## Development Commands

```bash
# Local development (requires Openclaw in /openclaw or OPENCLAW_ENTRY set)
npm run dev

# Production start
npm start

# Syntax check
npm run lint

# Local smoke test (requires Docker)
npm run smoke
```

## Docker Build & Local Testing

```bash
# Build the container (builds Openclaw from source)
docker build -t openclaw-railway-template .

# Run locally with volume
docker run --rm -p 8080:8080 \
  -e PORT=8080 \
  -e SETUP_PASSWORD=test \
  -e OPENCLAW_STATE_DIR=/data/.openclaw \
  -e OPENCLAW_WORKSPACE_DIR=/data/workspace \
  -v $(pwd)/.tmpdata:/data \
  openclaw-railway-template

# Access setup wizard
open http://localhost:8080/setup  # password: test
```

## Architecture

### Request Flow

1. **User → Railway → Wrapper (Express on PORT)** → routes to:
   - `/setup/*` → setup wizard (auth: Basic with `SETUP_PASSWORD` when set; returns 500 if unset)
   - All other routes → proxied to internal gateway (auth: Basic with `SETUP_PASSWORD` when set; returns 500 if unset)

2. **Wrapper → Gateway** (localhost:18789 by default)
   - HTTP/WebSocket reverse proxy via `http-proxy`
   - Automatically injects `Authorization: Bearer <token>` header

### Security architecture (wrapper auth, then token injection)

The intended design is **wrapper-level auth first**, then token injection:

```
Internet → Railway → Wrapper (auth: Basic SETUP_PASSWORD) → [if auth OK] inject gateway token & proxy → Gateway → Agent
```

- **Wrapper** enforces authentication (Basic auth with `SETUP_PASSWORD`) on all proxy and Control UI routes before any request reaches the gateway. No unauthenticated traffic is proxied.
- **Gateway token** is injected by the wrapper only after the incoming request has been authenticated. Clients never need to know the gateway token; the wrapper acts as a trusted intermediary.
- So the gateway’s token check is not “bypassed”—it still validates the injected token. The wrapper ensures only authenticated users can trigger those requests.

### Lifecycle States

1. **Unconfigured**: No `openclaw.json` exists
   - All non-`/setup` routes redirect to `/setup`
   - User completes setup wizard → runs `openclaw onboard --non-interactive`

2. **Configured**: `openclaw.json` exists
   - Wrapper spawns `openclaw gateway run` as child process
   - Waits for gateway to respond on multiple health endpoints
   - Proxies all traffic with injected bearer token

### Key Files

- **src/server.js** (thin entry point): Composes config, auth, gateway, onboarding, setup routes, proxy; Express app creation, `server.listen`, SIGTERM handler
- **src/gateway.js**: Gateway lifecycle — `startGateway`, `ensureGatewayRunning`, `restartGateway`, `waitForGatewayReady`, `clawArgs`, `getGatewayProcess`; session lock/data clearing, MCP health check
- **src/onboard.js**: Auto-onboarding — `autoOnboard`, `canAutoOnboard`, `buildOnboardArgs`, `resolveTelegramAndWriteUserMd`, `envFingerprintForOnboard`, `shouldReOnboardDueToEnvChange`
- **src/bootstrap.mjs**: Bootstrap OpenClaw config — patches `openclaw.json` (models, agents, tools, gateway, channels, hooks), writes mcporter MCP config, seeds workspace prompt files, ensures Senpi state file
- **src/routes/setup.js**: Setup wizard & API — `createSetupRouter()` returns Express router for `/setup`, `/setup/healthz`, `/setup/api/*`, `/setup/export`
- **src/routes/proxy.js**: Gateway proxy — HTTP proxy, Control UI HTML interception (token script injection), catch-all middleware, WebSocket upgrade handler
- **src/lib/config.js**: Environment and paths — constants, env-derived values, `PROVIDER_TO_AUTH_CHOICE`, `resolveEffectiveApiKey()`, `configPath()`, `isConfigured()`
- **src/lib/auth.js**: Auth helpers — `tokenLogSafe`, `secureCompare`, `resolveGatewayToken`, `createRequireSetupAuth`, `createCheckProxyAuth`
- **src/lib/runCmd.js**: `runCmd(cmd, args, opts)` — spawns process with env overrides, returns `{ code, output }`
- **src/lib/deviceAuth.js**: Auto-approval of pending loopback operator devices — burst+steady polling loop after gateway start
- **src/lib/models.js**: AI model catalog (`DESIRED_MODELS`), provider defaults (`PROVIDER_DEFAULTS`), provider-to-model mapping (`AI_PROVIDER_MODEL_MAP`)
- **src/lib/telegramId.js**: Telegram user-ID resolution via Bot API and caching — `resolveTelegramUserId`, `readCachedTelegramId`, `writeCachedTelegramId`, `readChatIdFromUserMd`
- **src/public/** (static assets for setup wizard):
  - **setup.html**: Setup wizard HTML structure
  - **styles.css**: Setup wizard styling (extracted from inline styles)
  - **setup-app.js**: Client-side JS for `/setup` wizard (vanilla JS, no build step)
- **Dockerfile**: Multi-stage build (builds Openclaw from source, installs wrapper deps)

### Environment Variables

**Required (for zero-touch):** `AI_PROVIDER`, `AI_API_KEY`, `TELEGRAM_BOT_TOKEN`, `SENPI_AUTH_TOKEN` (see README)

**Recommended:**

- `SETUP_PASSWORD` — when set, protects `/setup` and gateway/Control UI (/, /openclaw) with Basic auth. When **not** set, those routes are disabled (return 500) and a prominent startup warning is logged; the deployment is not publicly accessible for setup or Control UI.

**Recommended (Railway template defaults):**

- `OPENCLAW_STATE_DIR=/data/.openclaw` — config + credentials
- `OPENCLAW_WORKSPACE_DIR=/data/workspace` — agent workspace

**Optional:**

- `OPENCLAW_GATEWAY_TOKEN` — auth token for gateway (auto-generated if unset)
- `PORT` — wrapper HTTP port (default 8080)
- `INTERNAL_GATEWAY_PORT` — gateway internal port (default 18789)
- `INTERNAL_GATEWAY_HOST` — gateway internal host (default `127.0.0.1`)
- `GATEWAY_READY_TIMEOUT_MS` — how long to wait for gateway readiness (default `20000`)
- `OPENCLAW_ENTRY` — path to `entry.js` (default `/openclaw/dist/entry.js`)
- `OPENCLAW_NODE` — node binary for running openclaw CLI (default `node`)
- `OPENCLAW_CONFIG_PATH` — override config file location (default `${STATE_DIR}/openclaw.json`)
- `OPENCLAW_TEMPLATE_DEBUG` — set to `true` to enable `/setup/api/debug` endpoint
- `TELEGRAM_USERID` — alias for `TELEGRAM_USERNAME` (checked first)
- `SENPI_MCP_URL` — Senpi MCP endpoint (default `https://mcp.dev.senpi.ai/mcp`)
- `MCPORTER_CONFIG` — path to mcporter config (default `${STATE_DIR}/config/mcporter.json`)

### Authentication Flow

The wrapper manages a **two-layer auth scheme**:

1. **Setup wizard auth**: Basic auth with `SETUP_PASSWORD` via `createRequireSetupAuth()` (src/lib/auth.js)
2. **Proxy/Control UI auth**: Basic auth with `SETUP_PASSWORD` via `createCheckProxyAuth()` (src/lib/auth.js)
3. **Gateway auth**: Bearer token with multi-source resolution and automatic sync
   - **Token resolution order** — `resolveGatewayToken()` (src/lib/auth.js):
     1. `OPENCLAW_GATEWAY_TOKEN` env variable (highest priority)
     2. Persisted file at `${STATE_DIR}/gateway.token`
     3. Generate new random token (32 bytes hex) and persist
   - **Token synchronization**:
     - During onboarding: Synced to `openclaw.json` with verification (src/onboard.js `autoOnboard`, src/routes/setup.js `/api/run`)
     - Every gateway start: Synced to `openclaw.json` with verification (src/gateway.js `startGateway`)
     - Reason: Openclaw gateway reads token from config file, not from `--token` flag
   - **Token injection**:
     - HTTP requests: via `proxy.on("proxyReq")` event handler (src/routes/proxy.js)
     - WebSocket upgrades: via `proxy.on("proxyReqWs")` event handler (src/routes/proxy.js)
     - WebSocket `upgrade` event: direct `req.headers.authorization` replacement (src/routes/proxy.js `attachUpgrade`)

### Onboarding Process

**Manual setup** — via `/setup/api/run` (src/routes/setup.js):

1. Calls `openclaw onboard --non-interactive` via `buildOnboardArgs()` (src/onboard.js)
2. Syncs wrapper token, gateway config, and channel configs
3. Runs `bootstrapOpenClaw()` (src/bootstrap.mjs) and restarts gateway

**Auto-onboard** — via `autoOnboard()` (src/onboard.js):

1. Resolves Telegram user ID and writes USER.md (`resolveTelegramAndWriteUserMd`)
2. Calls `openclaw onboard --non-interactive` with env-derived auth provider and `--gateway-token` flag
3. **Syncs wrapper token to `openclaw.json`** (overwrites whatever `onboard` generated):
   - Sets `gateway.auth.token` to `OPENCLAW_GATEWAY_TOKEN` env variable
   - Verifies sync succeeded by reading config file back
   - Throws on mismatch
4. Writes channel configs (Telegram) directly via `openclaw config set --json`
5. Force-sets gateway config: token auth, loopback bind, port, allowInsecureAuth, dangerouslyDisableDeviceAuth, trustedProxies
6. Runs `bootstrapOpenClaw()` (src/bootstrap.mjs) — patches openclaw.json with models, hooks, tools config
7. Restarts gateway and writes env fingerprint for redeploy detection

**Re-onboard** — triggered on startup when `shouldReOnboardDueToEnvChange()` detects env fingerprint mismatch (src/server.js).

**Important**: Channel setup bypasses `openclaw channels add` and writes config directly because `channels add` is flaky across different Openclaw builds.

### Gateway Token Injection

The wrapper **always** injects the bearer token into proxied requests so browser clients don't need to know it:

- HTTP requests: via `proxy.on("proxyReq")` event handler (src/routes/proxy.js)
- WebSocket upgrades: via `proxy.on("proxyReqWs")` event handler (src/routes/proxy.js)
- WebSocket `upgrade`: via `attachUpgrade()` — replaces `req.headers.authorization` after Basic auth check (src/routes/proxy.js)

**Important**: Token injection for the HTTP proxy uses `http-proxy` event handlers (`proxyReq` and `proxyReqWs`) rather than direct `req.headers` modification. Direct header modification does not reliably work with WebSocket upgrades, causing intermittent `token_missing` or `token_mismatch` errors.

The Control UI at `/openclaw` works without the user knowing the gateway token — the wrapper fetches upstream HTML, injects a `<script>` that auto-fills the token into localStorage/cookie/input fields, and auto-clicks Connect (src/routes/proxy.js `controlUiHandler`).

### Backup Export

`GET /setup/export` (src/routes/setup.js):

- Creates a `.tar.gz` archive of `STATE_DIR` and `WORKSPACE_DIR`
- **Excludes secrets:** `gateway.token`, `openclaw.json`, `mcporter.json`, and `*.token` files are not included
- Audit log: each export request is logged with a warning

## Common Development Tasks

### Testing the setup wizard

1. Delete `${STATE_DIR}/openclaw.json` (or run Reset in the UI)
2. Visit `/setup` and complete onboarding
3. Check logs for gateway startup and channel config writes

### Testing authentication

- Setup wizard: Clear browser auth, verify Basic auth challenge
- Gateway: Remove `Authorization` header injection in `proxy.on("proxyReq")` (src/routes/proxy.js) and verify requests fail

### Debugging gateway startup

Check logs for:

- `[gateway] starting with command: ...` (src/gateway.js `startGateway`)
- `[gateway] ready at <endpoint>` (src/gateway.js `waitForGatewayReady`)
- `[gateway] failed to become ready after 20000ms` (src/gateway.js `waitForGatewayReady`)
- `[gateway] MCP health check: OK/FAIL` (src/gateway.js `checkMcpHealth`)
- `[deviceAuth] Auto-approval loop started` (src/lib/deviceAuth.js)

If gateway doesn't start:

- Verify `openclaw.json` exists and is valid JSON
- Check `STATE_DIR` and `WORKSPACE_DIR` are writable
- Ensure bearer token is set in config
- Check for stale session locks (auto-cleared on startup by `clearStaleSessionLocks` in src/gateway.js)

### Modifying onboarding args

Edit `buildOnboardArgs()` in src/onboard.js to add new CLI flags or auth providers. Update `PROVIDER_TO_AUTH_CHOICE` in src/lib/config.js for new provider mappings.

### Adding new channel types

1. Add channel-specific fields to `/setup` HTML (src/public/setup.html)
2. Add config-writing logic in `/setup/api/run` handler (src/routes/setup.js)
3. Update client JS to collect the fields (src/public/setup-app.js)
4. Add auto-onboard channel config in `autoOnboard()` (src/onboard.js)
5. Update bootstrap patch in `patchOpenClawJson()` (src/bootstrap.mjs)

### Modifying models or provider defaults

Edit src/lib/models.js — `DESIRED_MODELS` for the allowlist, `PROVIDER_DEFAULTS` for env-var-based defaults, `AI_PROVIDER_MODEL_MAP` for `AI_PROVIDER` fallback mapping. Bootstrap merges these into `openclaw.json` on every startup.

## Railway Deployment Notes

- Template must mount a volume at `/data`
- **Recommended:** set `SETUP_PASSWORD` in Railway Variables so `/setup` and Control UI (/, /openclaw) are accessible. If unset, those routes return 500 and a startup warning is logged.
- Public networking must be enabled (assigns `*.up.railway.app` domain)
- Openclaw version is pinned via Docker build arg `OPENCLAW_VERSION` (default: `v2026.2.22`). The Dockerfile falls back to `v2026.2.22` if the arg is not set. For older versions (e.g. 2026.2.12), some features like `tools.fs` are not supported and are stripped by bootstrap. See [releases](https://github.com/openclaw/openclaw/releases).

## Serena Semantic Coding

This project has been onboarded with **Serena** (semantic coding assistant via MCP). Comprehensive memory files are available covering:

- Project overview and architecture
- Tech stack and codebase structure
- Code style and conventions
- Development commands and task completion checklist
- Quirks and gotchas

**When working on tasks:**

1. Check `mcp__serena__check_onboarding_performed` first to see available memories
2. Read relevant memory files before diving into code (e.g., `mcp__serena__read_memory`)
3. Use Serena's semantic tools for efficient code exploration:
   - `get_symbols_overview` - Get high-level file structure without reading entire file
   - `find_symbol` - Find classes, functions, methods by name path
   - `find_referencing_symbols` - Understand dependencies and usage
4. Prefer symbolic editing (`replace_symbol_body`, `insert_after_symbol`) for precise modifications

This avoids repeatedly reading large files and provides instant context about the project.

## Quirks & Gotchas

1. **Gateway token must be stable across redeploys** → Always set `OPENCLAW_GATEWAY_TOKEN` env variable in Railway (highest priority); token is synced to `openclaw.json` during onboarding (src/onboard.js `autoOnboard`) and on every gateway start (src/gateway.js `startGateway`) with verification. This is required because `openclaw onboard` generates its own random token and the gateway reads from config file, not from `--token` CLI flag. Sync failures throw errors and prevent gateway startup.
2. **Channels are written via `config set --json`, not `channels add`** → avoids CLI version incompatibilities
3. **Gateway readiness check polls multiple endpoints** (`/openclaw`, `/`, `/health`) → some builds only expose certain routes (src/gateway.js `waitForGatewayReady`)
4. **Discord bots require MESSAGE CONTENT INTENT** → document this in setup wizard
5. **Gateway spawn pipes stdout/stderr** → logs appear in wrapper output (src/gateway.js `startGateway`); stderr tail (last 4KB) is captured and logged on non-zero exit
6. **WebSocket auth requires proxy event handlers** → Direct `req.headers` modification doesn't work for WebSocket upgrades with http-proxy; must use `proxyReqWs` event (src/routes/proxy.js) to reliably inject Authorization header
7. **Control UI and headless internal clients** → We set `gateway.controlUi.allowInsecureAuth=true` (Control UI behind proxy) and `gateway.controlUi.dangerouslyDisableDeviceAuth=true` (headless: no device to pair). The latter is required so internal clients (Telegram provider, cron, session WS) connecting from 127.0.0.1 with the token from config are not rejected with `code=1008 reason=connect failed` / "pairing required". Both are set in src/bootstrap.mjs, src/onboard.js, src/gateway.js, and src/routes/setup.js post-onboard.
8. **"pairing required" / "connect failed" (1008)** → If you still see this after the above: check that `openclaw.json` has `gateway.controlUi.dangerouslyDisableDeviceAuth: true` and restart the gateway (redeploy or restart the process). The wrapper also runs an auto-approval loop (src/lib/deviceAuth.js) that continuously approves pending loopback operator devices. Wrapper logs `[ws-upgrade]` only for browser→wrapper→gateway; internal client failures appear in gateway logs as `[ws] closed before connect ... code=1008 reason=connect failed`.
9. **`[tools] read failed: ENOENT ... access '/openclaw/src/...'`** → The agent tried to read a path outside the workspace (e.g. OpenClaw source). Note: `tools.fs.workspaceOnly` is NOT supported on OpenClaw 2026.2.12 — bootstrap strips this field if present (src/bootstrap.mjs). For newer builds, enable it manually.
10. **`[telegram] sendChatAction failed: Network request for 'sendChatAction' failed!`** → Telegram API call (e.g. "typing…" indicator) failed. Usually transient (network, rate limit, or egress). If persistent, check TELEGRAM_BOT_TOKEN and egress to api.telegram.org. Chat delivery can still work when sendChatAction fails.
11. **Session data cleared on every gateway restart** → `clearAllSessions()` (src/gateway.js) deletes all session files to prevent stale context errors like "No tool call found for function call output".
12. **Device auth auto-approval loop** → After gateway becomes ready, `startAutoApprovalLoop()` (src/lib/deviceAuth.js) polls `openclaw devices list` with a burst+steady schedule (aggressive for ~60s, then once per minute) and auto-approves any pending loopback operator devices.
13. **`TELEGRAM_USERID` takes precedence** → src/lib/config.js reads `TELEGRAM_USERID` first, then falls back to `TELEGRAM_USERNAME`. Both are accepted as @username or numeric chat ID.
14. **Provider-specific API key fallback** → If `AI_API_KEY` is not set, `resolveEffectiveApiKey()` (src/lib/config.js) checks provider-specific env vars (e.g. `ANTHROPIC_API_KEY`, `VENICE_API_KEY`) based on `AI_PROVIDER`.
15. **Managed workspace files overwritten on deploy** → AGENTS.md, SOUL.md, BOOTSTRAP.md, TOOLS.md in the workspace are always overwritten from `/opt/workspace-defaults` by `seedWorkspaceFiles()` (src/bootstrap.mjs). Non-managed files (e.g. USER.md, MEMORY.md) are preserved.
16. **Bootstrap creates Senpi state file** → `ensureSenpiStateFile()` (src/bootstrap.mjs) creates `~/.config/senpi/state.json` with `{}` if absent, so the agent's BOOTSTRAP.md can read it without ENOENT.
17. **MCP health check after gateway ready** → `checkMcpHealth()` (src/gateway.js) runs a fire-and-forget `mcporter call senpi.user_get_me` to verify MCP connectivity.
