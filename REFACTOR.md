# Refactoring: from monolith to maintainable modules

The main server logic used to live in a single large file (`server.js`), which was hard to understand, modify, and test. This document describes the current (fully modularized) state.

## Current state (refactoring complete)

`server.js` is now a **thin composition root** (~165 lines) that imports and wires together all modules. All business logic lives in dedicated modules.

### Shared libraries (`src/lib/`)

- **`src/lib/config.js`** – Environment and paths
  - Constants: `PORT`, `STATE_DIR`, `WORKSPACE_DIR`, `GATEWAY_TARGET`, `INTERNAL_GATEWAY_PORT`, `GATEWAY_READY_TIMEOUT_MS`, `OPENCLAW_ENTRY`, `OPENCLAW_NODE`
  - Env-derived: `SETUP_PASSWORD`, `TELEGRAM_BOT_TOKEN`, `TELEGRAM_USERNAME` (reads `TELEGRAM_USERID` first), `AI_PROVIDER`, `AI_API_KEY`, `PROVIDER_TO_AUTH_CHOICE`, `PROVIDERS_WITHOUT_API_KEY`, `DEBUG`, `TELEGRAM_CHAT_ID_FILE`
  - Helpers: `stripBearer()`, `resolveEffectiveApiKey()`, `configPath()`, `isConfigured()`, `ensureWritableDirs()`
  - **Testable:** Pure functions and values; no gateway or HTTP.

- **`src/lib/auth.js`** – Auth and token helpers
  - `tokenLogSafe(token)` – Safe fingerprint for logs
  - `secureCompare(a, b)` – Timing-safe string comparison
  - `resolveGatewayToken(stateDir?)` – Resolve or generate gateway token (reads env + file, may write file)
  - `createRequireSetupAuth(password)` – Express middleware factory for /setup Basic auth
  - `createCheckProxyAuth(password)` – Auth check factory for proxy/Control UI routes
  - **Testable:** `tokenLogSafe` and `secureCompare` are pure; `resolveGatewayToken` can be tested with a temp dir.

- **`src/lib/runCmd.js`** – Run a command with env
  - `runCmd(cmd, args, opts?)` – Spawns process, returns `{ code, output }`; injects `OPENCLAW_STATE_DIR` and `OPENCLAW_WORKSPACE_DIR` from config.
  - **Testable:** Can mock `child_process.spawn` or run real commands in tests.

- **`src/lib/deviceAuth.js`** – Auto-approval of loopback operator devices
  - `autoApprovePendingOperatorDevices()` – Lists pending devices via CLI and approves loopback operators
  - `startAutoApprovalLoop()` – Persistent polling: burst phase (~60s aggressive), then steady (once per minute)
  - `stopAutoApprovalLoop()` – Stop the polling loop
  - **Testable:** Can mock `runCmd` to simulate device lists.

- **`src/lib/models.js`** – AI model catalog and provider defaults
  - `DESIRED_MODELS` – Comprehensive model allowlist (Anthropic, OpenAI, Google, xAI, Groq, Mistral, Together, Z.AI, Moonshot, Venice, MiniMax, NVIDIA, OpenRouter, OpenCode, HuggingFace, Bedrock)
  - `PROVIDER_DEFAULTS` – Map provider-specific env vars to default models
  - `AI_PROVIDER_MODEL_MAP` – Fallback: map `AI_PROVIDER` value to default model
  - **Testable:** Pure data exports.

- **`src/lib/telegramId.js`** – Telegram user-ID resolution and caching
  - `readCachedTelegramId()` – Read cached numeric ID from disk
  - `readChatIdFromUserMd()` – Read numeric ID from USER.md
  - `writeCachedTelegramId(id)` – Write numeric ID to cache file
  - `resolveTelegramUserId(botToken, username)` – Resolve @username to numeric ID via Bot API getUpdates
  - **Testable:** Sync functions are pure; async resolution can be tested with API mocks.

### Core modules (`src/`)

- **`src/gateway.js`** – Gateway lifecycle
  - `startGateway(gatewayToken)` – Token sync, spawn, pipe stdout/stderr, capture stderr tail
  - `ensureGatewayRunning(gatewayToken)` – Idempotent start + wait for ready + start auto-approval loop + MCP health check
  - `restartGateway(gatewayToken)` – Kill + pkill + sleep + ensureGatewayRunning
  - `waitForGatewayReady(opts?)` – Poll multiple endpoints with configurable timeout
  - `clawArgs(args)` – Prepend `OPENCLAW_ENTRY` to CLI args
  - `getGatewayProcess()` – For SIGTERM cleanup
  - Internal: `clearStaleSessionLocks()`, `clearAllSessions()`, `checkMcpHealth()`

- **`src/onboard.js`** – Auto-onboarding and Telegram USER.md resolution
  - `autoOnboard(gatewayToken)` – Full auto-onboard flow from env vars (resolve Telegram, onboard, sync config, channels, bootstrap, restart gateway, write fingerprint)
  - `canAutoOnboard()` – Check if env vars allow auto-onboarding
  - `buildOnboardArgs(payload, gatewayToken)` – Build CLI args for `openclaw onboard --non-interactive`
  - `resolveTelegramAndWriteUserMd()` – Resolve Telegram user and write USER.md (preserves existing sections)
  - `envFingerprintForOnboard()` – SHA256 of env vars for redeploy detection
  - `shouldReOnboardDueToEnvChange()` – Compare stored vs current fingerprint
  - `isOnboardingInProgress()`, `AUTO_ONBOARD_FINGERPRINT_FILE`

- **`src/bootstrap.mjs`** – Bootstrap OpenClaw configuration
  - `bootstrapOpenClaw()` – Orchestrator: ensure dirs, MEMORY.md, skills, Senpi state, MCP config, workspace files, patch openclaw.json
  - Internal: `patchOpenClawJson()` (agents, tools, gateway, channels, hooks, models), `writeMcporterConfig()`, `seedWorkspaceFiles()`, `ensureSenpiStateFile()`

### Route modules (`src/routes/`)

- **`src/routes/setup.js`** – Setup wizard & API
  - `createSetupRouter()` – Returns Express router for `/setup`, `/setup/healthz`, `/setup/api/*`, `/setup/export`
  - Routes: GET `/`, `/app.js`, `/styles.css`, `/api/status`, `/api/gateway-token`, `/api/debug`; POST `/api/run`, `/api/reset`, `/api/pairing/approve`, `/api/senpi-token`

- **`src/routes/proxy.js`** – Gateway proxy
  - `controlUiMiddleware` / `controlUiHandler` – Control UI HTML interception + token script injection
  - `catchAllMiddleware` – Require auth, proxy to gateway (or redirect to /setup during onboarding)
  - `attachUpgrade(server)` – WebSocket upgrade handler with auth + token injection

### Entry point (`src/server.js`)

Thin composition root (~165 lines):
- Mounts setup router, Control UI routes, catch-all proxy
- On `server.listen`: runs re-onboard / auto-onboard / bootstrap+gateway based on state
- `attachUpgrade(server)` for WebSocket
- SIGTERM handler: stops auto-approval loop, kills gateway

## Testing

- **Unit tests** (e.g. with Node's `node:test` or Jest):
  - `lib/config.js`: `configPath()`, `isConfigured()` with a temp dir; `stripBearer()`; `resolveEffectiveApiKey()` with mocked `process.env`.
  - `lib/auth.js`: `tokenLogSafe()`, `secureCompare()`; `resolveGatewayToken()` with temp dir and env mocks.
  - `lib/runCmd.js`: `runCmd()` with mocked `child_process.spawn`.
  - `lib/models.js`: Verify `DESIRED_MODELS`, `PROVIDER_DEFAULTS`, `AI_PROVIDER_MODEL_MAP` structure.
  - `lib/telegramId.js`: `readCachedTelegramId()`, `writeCachedTelegramId()` with temp dir; `resolveTelegramUserId()` with API mocks.
  - `lib/deviceAuth.js`: `autoApprovePendingOperatorDevices()` with mocked `runCmd`.
- **Integration tests**: Start the app (or a test app that only mounts a subset of routes), hit `/setup/healthz`, proxy to a stub gateway, etc.
