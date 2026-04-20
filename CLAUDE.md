# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

- `npm run build` — compile TypeScript (`src/` → `dist/`). The `prepare`/`prepublishOnly` hooks also run this.
- `npm test` — run vitest once. Use `npm run test:watch` for watch mode.
- Run a single test file: `npx vitest run src/reply-all-helpers.test.ts`
- Run tests matching a name: `npx vitest run -t "reply all recipient"`
- `npm start` — run the built server over stdio (`dist/index.js`).
- `npm run auth` — OAuth flow. Supports `--scopes=<comma-list>` (see `src/scopes.ts` for names). Default: `gmail.modify,gmail.settings.basic`. Credentials land in `~/.gmail-mcp/credentials.json`.
- Evals: `OPENAI_API_KEY=... npx mcp-eval src/evals/evals.ts src/index.ts` (loads a client against `src/index.ts` directly — no rebuild required).

Note: `tsconfig.json` excludes `*.test.ts` from the build; tests only run through vitest, never through `tsc`.

## Architecture

Local stdio MCP server exposing Gmail via the Google APIs. One long-running process per Claude client, launched on demand via `npx`.

**Entry point — `src/index.ts`** (single ~1500-line file; intentional, matches upstream shape):
- Parses CLI args (`auth` subcommand, `--scopes`, optional callback URL).
- `loadCredentials()` reads `~/.gmail-mcp/gcp-oauth.keys.json` + `credentials.json` and recovers the scopes the user was actually authorized with (falls back to `DEFAULT_SCOPES`).
- `authenticate()` runs a local `http://localhost:3000/oauth2callback` server and opens the browser.
- Registers two handlers: `ListToolsRequestSchema` returns only tools whose `scopes` are satisfied by the current credentials; `CallToolRequestSchema` dispatches by tool name into a big switch. Tools not in scope return an "is not available … re-authenticate" error rather than being hidden silently after listing.

**Scope-gated tool registry — `src/tools.ts`**: `toolDefinitions` is the single source of truth. Each entry has `{ name, description, schema (zod), scopes: string[], annotations }`. `scopes` is OR-semantics — any one of them grants access. `toMcpTools()` converts to MCP wire format via `zod-to-json-schema`. When adding a tool: add a zod schema + an entry here, then a `case` in the `CallToolRequestSchema` switch in `index.ts`.

**Scope model — `src/scopes.ts`**: Short names (`gmail.readonly`, `gmail.modify`, `gmail.compose`, `gmail.send`, `gmail.labels`, `gmail.settings.basic`) map to full Google API URLs. `gmail.modify` is a superset of `gmail.readonly`; don't request both. `hasScope()` normalizes URLs back to short names before comparing, so either form works in `toolDefinitions`.

**Feature modules**:
- `label-manager.ts` / `filter-manager.ts` — thin wrappers over `googleapis` Gmail label and filter endpoints, plus the `filterTemplates` used by `create_filter_from_template`.
- `reply-all-helpers.ts` — recipient-list and threading-header construction (`buildReplyAllRecipients`, `buildReferencesHeader`, `addRePrefix`). Pure functions; heavily tested.
- `utl.ts` — `createEmailMessage` (raw RFC822) and `createEmailWithNodemailer` (attachment path). `send_email` routes through nodemailer when attachments are present, otherwise builds the MIME message directly.
- `email-export.ts` — format converters used by `download_email` (json/eml/txt/html).

**OAuth state**: Global variables `oauth2Client` and `authorizedScopes` in `index.ts`. Config paths are overridable via `GMAIL_OAUTH_PATH` / `GMAIL_CREDENTIALS_PATH` env vars (used by the Docker setup).

## Branch Workflow

Two-branch model: `main` (stable) and `experimental` (staging). PRs and local changes always land on `experimental` first; promote to `main` only after user confirmation. Always end sessions on `experimental`. After every push, verify CI with `gh run list --branch <branch> --limit 1` and fix failures immediately. Full SOP: `.claude/skills/pr-review-sop/SKILL.md`.

## CI README Check

CI (`.github/workflows/ci.yml`) fails any push/PR that modifies `src/**` without also touching `README.md`. To bypass for genuine non-docs changes (deps, CI config, refactors with no user-visible effect), put `[skip-readme]` or `[no-readme]` in the commit message (push) or PR title (PR). Merge commits are auto-skipped. When CI fails this check, it opens/appends to a `readme-gap` issue.

## PR Review & Security Audit

Every PR gets a mandatory security audit before being presented to the user — see `.claude/skills/pr-review-sop/SKILL.md` for the full checklist. Approved PRs are retargeted to `experimental` and merged via `gh pr merge --merge` (purple badge — never manually close).

**Local threat model**: This is a local stdio MCP server. The LLM client already has Bash and Write. Do **not** flag path traversal, filename injection, local XSS in exported files, or symlink following as security issues — they're not exploitable in this context. Credential leaks to third parties, network-exposed endpoints, and supply-chain issues *are* real and must be flagged.
