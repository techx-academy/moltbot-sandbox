# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Cloudflare Worker that runs [Moltbot](https://molt.bot/) personal AI assistant in a [Cloudflare Sandbox](https://developers.cloudflare.com/sandbox/) container. Provides web UI at `/`, admin UI at `/_admin/`, API endpoints at `/api/*`, debug endpoints at `/debug/*`, and a CDP shim for browser automation at `/cdp/*`.

**Important:** The CLI tool is named `clawdbot` internally (upstream hasn't renamed), so CLI commands and config paths use that name (e.g., `~/.clawdbot/clawdbot.json`).

## Commands

```bash
npm test              # Run tests (vitest)
npm run test:watch    # Run tests in watch mode
npm run build         # Build worker + client
npm run deploy        # Build and deploy to Cloudflare
npm run dev           # Vite dev server for admin UI
npm run start         # wrangler dev (local worker)
npm run typecheck     # TypeScript type checking
```

## Architecture

```
Browser → Cloudflare Worker (Hono) → Cloudflare Sandbox Container
                                          ├── Moltbot Gateway (port 18789)
                                          ├── R2 Storage (mounted at /data/moltbot)
                                          └── Optional: Chat integrations
```

**Key files:**
- `src/index.ts` - Main Hono app, route mounting, sandbox lifecycle
- `Dockerfile` - Container image (Node 22 + moltbot)
- `start-moltbot.sh` - Container startup: R2 restore, config, gateway launch
- `moltbot.json.template` - Default moltbot config (copied to `~/.clawdbot/clawdbot.json`)
- `wrangler.jsonc` - Worker + container configuration

## Code Organization

- `src/auth/` - Cloudflare Access JWT verification and middleware
- `src/gateway/` - Container management: process lifecycle, env vars, R2 mounting/sync
- `src/routes/` - Route handlers (api.ts, admin-ui.ts, debug.ts, cdp.ts, public.ts)
- `src/client/` - React admin UI (Vite build)

Tests are colocated with source files (`*.test.ts`).

## Key Patterns

### CLI Commands
Always include `--url ws://localhost:18789`:
```typescript
sandbox.startProcess('clawdbot devices list --json --url ws://localhost:18789')
```
CLI commands take 10-15 seconds due to WebSocket overhead.

### Success Detection
The CLI outputs "Approved" with capital A. Use case-insensitive checks:
```typescript
stdout.toLowerCase().includes('approved')
```

### Environment Variables
- `DEV_MODE` - Skips CF Access auth AND bypasses device pairing
- `DEBUG_ROUTES` - Enables `/debug/*` routes
- API provider options (one required):
  - `ANTHROPIC_API_KEY` + optional `ANTHROPIC_BASE_URL`
  - `OPENAI_API_KEY` + optional `OPENAI_BASE_URL` (for OpenAI-compatible APIs)
  - `AI_GATEWAY_API_KEY` + `AI_GATEWAY_BASE_URL` (for Cloudflare AI Gateway)
- See `src/types.ts` for full `MoltbotEnv` interface
- Container env vars are mapped in `src/gateway/env.ts`

### R2 Storage Gotchas
- Mounted via s3fs at `/data/moltbot`
- Use `rsync -r --no-times` (s3fs doesn't support timestamps)
- Check mount status with `mount | grep s3fs`, not error messages
- Never `rm -rf /data/moltbot/*` - that's the R2 bucket contents

### Docker Image Caching
When changing `moltbot.json.template` or `start-moltbot.sh`, bump the cache bust comment in `Dockerfile`:
```dockerfile
# Build cache bust: 2026-01-26-v10
```

### Moltbot Config Schema
- `agents.defaults.model` must be `{ "primary": "model/name" }` not a string
- `gateway.mode` must be `"local"` for headless operation
- No `webchat` channel - Control UI is served automatically

## Adding New Functionality

### New API Endpoint
1. Add route handler in `src/routes/api.ts`
2. Add types in `src/types.ts` if needed
3. Update `src/client/api.ts` if frontend needs it
4. Add tests

### New Environment Variable
1. Add to `MoltbotEnv` in `src/types.ts`
2. If passed to container, add to `buildEnvVars()` in `src/gateway/env.ts`
3. Update `.dev.vars.example`
4. Document in README.md secrets table

## Local Development

```bash
npm install
cp .dev.vars.example .dev.vars
# Edit .dev.vars with ANTHROPIC_API_KEY and DEV_MODE=true
npm run start
```

WebSocket connections may fail with `wrangler dev`. Deploy to Cloudflare for full functionality testing.
