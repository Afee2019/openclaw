# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What is OpenClaw

OpenClaw is a personal AI assistant platform. It runs a **Gateway** (WebSocket control plane) that bridges messaging channels (WhatsApp, Telegram, Slack, Discord, Signal, iMessage, Microsoft Teams, Google Chat, Matrix, etc.) to AI models. 

It includes native apps (macOS/iOS/Android), a web control UI, a CLI, and an extensible skills/plugin system.

## Tech Stack

- **Language**: TypeScript (ESM), strict mode, legacy decorators
- **Runtime**: Node 22+ (Bun also supported for dev/scripts)
- **Package Manager**: pnpm 10.x (via corepack) — **never use npm**
- **Bundler**: tsdown (main), Vite (UI)
- **Testing**: Vitest (V8 coverage, forked pool)
- **Linting**: oxlint (type-aware) + oxfmt
- **UI Framework**: Lit 3 (web components, legacy decorators)
- **Native Apps**: SwiftUI (macOS/iOS), Kotlin (Android)
- **Database**: SQLite with sqlite-vec (embeddings)

## Common Commands

```bash
pnpm install                  # Install dependencies
pnpm build                    # Full build (tsdown + plugin-sdk + UI assets)
pnpm check                    # Type-check (tsgo) + lint (oxlint) + format (oxfmt)
pnpm test                     # Unit + extension + gateway tests (parallel via scripts/test-parallel.mjs)
pnpm test:coverage            # Tests with V8 coverage report
pnpm test:e2e                 # E2E tests (vitest.e2e.config.ts)
pnpm test:live                # Live integration tests (requires API keys: OPENCLAW_LIVE_TEST=1)
pnpm test:watch               # Vitest in watch mode
pnpm dev                      # Dev server with hot reload
pnpm gateway:dev              # Gateway dev mode (skips channel init)
pnpm lint:fix                 # Auto-fix lint + format issues
pnpm ui:dev                   # Control UI dev server (Vite, port 5173)
pnpm ui:build                 # Build control UI
pnpm protocol:check           # Validate protocol schema + Swift codegen are up to date
```

### Running a single test

```bash
# By file path
pnpm vitest run src/path/to/file.test.ts

# By name pattern
pnpm vitest run -t "test name pattern"

# Watch a specific file
pnpm vitest src/path/to/file.test.ts
```

### Pre-PR gate (what CI runs)

```bash
pnpm build && pnpm check && pnpm test
```

## Project Architecture

```
src/                        Core engine (~2600 TS files)
├── gateway/                WebSocket server, protocol, auth, state machine
├── agents/                 Agent orchestration, auth profiles, tool management
├── channels/               Channel abstraction layer
│   ├── discord/            Discord integration
│   ├── telegram/           Telegram bot
│   ├── slack/              Slack workspace adapter
│   ├── signal/             Signal protocol
│   ├── imessage/           iMessage (macOS only)
│   ├── web/                WebChat
│   └── plugins/            ~40 channel plugin adapters
├── routing/                Bindings, session keys, route resolution
├── commands/               CLI command implementations
├── cli/                    CLI wiring, profile management
├── config/                 Config schema (Zod), validation, persistence
├── infra/                  Infrastructure (Bonjour, Tailscale, ports, archiving)
├── media/                  Audio/image processing pipeline
├── browser/                Playwright browser automation
├── hooks/                  Lifecycle hooks
├── cron/                   Scheduled tasks
├── plugin-sdk/             Public extension API (exported as openclaw/plugin-sdk)
├── canvas-host/            Agent-editable HTML canvas (separate WS port 18793)
└── acp/                    Agent Control Protocol

extensions/                 Channel plugins as workspace packages (36+)
skills/                     Bundled skills (54+), each with own package.json
ui/                         Lit web components control dashboard (Vite)
apps/
├── macos/                  SwiftUI macOS app
├── ios/                    SwiftUI iOS app
├── android/                Kotlin Android app
└── shared/                 OpenClawKit Swift package
docs/                       Mintlify documentation site
scripts/                    Build, test, deployment scripts (70+)
test/                       Shared test utilities, fixtures, mocks
```

### Key Architectural Concepts

**Gateway model**: Single long-lived WebSocket server (default port 18789). All messaging channels, native apps, and the web UI connect here. Protocol uses request/response with unique IDs + event streaming.

**Plugin/extension system**: Core channel adapters in `src/channels/plugins/`, external plugins in `extensions/` (pnpm workspace packages). Plugin SDK at `src/plugin-sdk/index.ts`. Plugin deps go in the extension's own `package.json`, not root. Use `devDependencies` or `peerDependencies` for `openclaw` (runtime resolves via jiti alias).

**Skills**: Domain-specific capabilities in `skills/`. Each skill is self-contained with its own `package.json` and can include tools, models, and automations.

**Multi-agent**: Multiple agents can run concurrently with auth profile failover (round-robin, last-good). Identity/personality per agent.

**Config**: Zod schemas in `src/config/types*.ts`. User config at `~/.openclaw/config.json`.

## Coding Conventions

- **TypeScript ESM** with strict typing. Avoid `any`.
- **Formatting/linting**: oxlint + oxfmt. Run `pnpm check` before commits.
- **File size**: aim for under ~500 LOC; split when it improves clarity.
- **Tests**: colocated as `*.test.ts`. E2E: `*.e2e.test.ts`. Live: `*.live.test.ts`.
- **Coverage thresholds**: 70% lines/functions/statements, 55% branches.
- **Naming**: "OpenClaw" for product/docs headings; `openclaw` for CLI/package/paths/config keys.
- **Commits**: use `scripts/committer "<msg>" <file...>` for scoped staging. Concise, action-oriented messages (e.g. `CLI: add verbose flag to send`).
- **Pre-commit hooks**: installed via `pnpm prepare` (git-hooks/pre-commit runs oxlint --fix + oxfmt --write on staged files).

### Control UI (Lit) specifics

Use **legacy decorators** (not standard). Rollup does not support `accessor` fields:

```ts
@state() foo = "bar";
@property({ type: Number }) count = 0;
```

Root `tsconfig.json` has `experimentalDecorators: true` + `useDefineForClassFields: false`.

### Tool schema guardrails

- No `Type.Union` in tool input schemas (no `anyOf`/`oneOf`/`allOf`).
- Use `stringEnum`/`optionalStringEnum` (Type.Unsafe enum) for string lists.
- Use `Type.Optional(...)` instead of `... | null`.
- Avoid raw `format` property names in tool schemas (reserved keyword in some validators).

## Channel Development

When refactoring shared channel logic (routing, allowlists, pairing, command gating, onboarding), consider **all** built-in + extension channels:
- Core: `src/telegram`, `src/discord`, `src/slack`, `src/signal`, `src/imessage`, `src/web`, `src/channels`
- Extensions: `extensions/*` (msteams, matrix, zalo, voice-call, etc.)

When adding a new connection/channel, update every UI surface and docs (macOS app, web UI, mobile if applicable, onboarding/overview docs).

## Docs (Mintlify)

- Hosted at docs.openclaw.ai. Internal links: root-relative, no `.md`/`.mdx` suffix (e.g. `[Config](/configuration)`).
- Avoid em dashes/apostrophes in headings (break Mintlify anchors).
- `docs/zh-CN/**` is auto-generated — do not edit unless explicitly asked.
- Docs content must be generic: no personal device names/hostnames; use placeholders.

## Multi-Agent Safety Rules

- Do **not** create/apply/drop `git stash` entries unless explicitly requested.
- Do **not** create/remove/modify `git worktree` or switch branches unless explicitly requested.
- When committing, scope to your changes only (unless told "commit all").
- `git pull --rebase` is OK when pushing; never discard other agents' work.
- When you see unrecognized files, keep going and focus on your changes.

## Important Constraints

- **GORM AutoMigrate is disabled**. Use manual migration scripts.
- Patched dependencies (`pnpm.patchedDependencies`) must use exact versions (no `^`/`~`).
- Patching dependencies requires explicit approval.
- Never update the Carbon dependency.
- Never send streaming/partial replies to external messaging channels (WhatsApp, Telegram, etc.) — only final replies.
- CLI progress: use `src/cli/progress.ts` (osc-progress + @clack/prompts spinner). Don't hand-roll spinners.
- Status output: use `src/terminal/table.ts` for tables + ANSI-safe wrapping.
- Colors: use shared CLI palette in `src/terminal/palette.ts` (no hardcoded colors).
- Version numbers: do not change without explicit consent.

## Also Read

- `AGENTS.md` — Full operational runbook (VM ops, release flow, 1Password publish, etc.)
- `CONTRIBUTING.md` — Contributor guidelines and PR process
- `docs/testing.md` — Full testing strategy
- `docs/concepts/architecture.md` — Gateway architecture deep-dive
