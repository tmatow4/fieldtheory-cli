# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

Field Theory CLI (`ft`) — a local-first tool for syncing and querying X/Twitter bookmarks. All data stored in `~/.ft-bookmarks/`. Published to npm as `fieldtheory`.

## Commands

```bash
npm run build          # Compile TypeScript → dist/
npm run dev            # Run via tsx (no build needed)
npm run test           # Run all tests (Node native test runner)
npm run start          # Run compiled dist/cli.js

# Run a single test file
npx tsx --test tests/bookmarks-db.test.ts
```

## Architecture

Single CLI app built with Commander.js. ES modules throughout, Node.js 20+.

### Two Sync Modes

| Mode | Command | Auth | Platform | Speed |
|------|---------|------|----------|-------|
| **GraphQL** (default) | `ft sync` | Chrome session cookies via macOS Keychain | macOS only | Fast |
| **OAuth API** | `ft sync --api` | OAuth 2.0 PKCE token | All platforms | Rate-limited |

GraphQL extracts `ct0`/`auth_token` from Chrome's encrypted cookie DB. OAuth uses Twitter API v2 with tokens stored in `oauth-token.json` (chmod 600).

### Data Flow

```
Chrome cookies or OAuth token
        ↓
  GraphQL API / Twitter API v2
        ↓
  bookmarks.jsonl  ← source of truth (append-only, deduplicated)
        ↓
  ft index
        ↓
  bookmarks.db  ← SQLite FTS5 index (rebuildable from JSONL)
        ↓
  ft classify (regex or LLM)
        ↓
  Search / List / Viz / Stats
```

**Key invariant:** JSONL is the source of truth. The SQLite index can always be rebuilt from it via `ft index --force`. Classifications are stored in the DB and preserved across re-indexes unless `--force` is used.

### Key Modules

| Module | Role |
|--------|------|
| `src/cli.ts` | All command definitions. Uses `safe()` wrapper for async error handling. `bin/ft.mjs` → no-args shows dashboard, with-args routes to Commander. |
| `src/graphql-bookmarks.ts` | GraphQL sync with retry/backoff, checkpoint every 25 pages, resumable via backfill state file |
| `src/bookmarks.ts` | OAuth API sync with pagination |
| `src/bookmarks-db.ts` | FTS5 schema (version 3), search with BM25 ranking, list with filters, stats, classification storage |
| `src/db.ts` | Thin WASM SQLite loader (sql.js-fts5). No persistent connection — read/query/write pattern. |
| `src/bookmark-classify.ts` | Regex classifier: 7 categories (tool, security, technique, launch, research, opinion, commerce) |
| `src/bookmark-classify-llm.ts` | LLM classifier: detects `claude` or `codex` CLI, batches of 50, sanitizes untrusted tweet text in `<tweet_text>` tags |
| `src/chrome-cookies.ts` | macOS Keychain → Chrome cookie DB decryption (AES-128-CBC, handles v24+ format) |
| `src/xauth.ts` | OAuth 2.0 PKCE flow with localhost:3000 callback server |

### Testing

Tests use **Node's native `node:test` module** with `node:assert/strict`. No external test framework.

```typescript
import test from 'node:test';
import assert from 'node:assert/strict';

test('description', async () => {
  const result = await functionUnderTest();
  assert.equal(result.count, expected);
});
```

### Dependencies

All pure JavaScript/WASM — no native bindings:
- `commander` — CLI framework
- `sql.js` + `sql.js-fts5` — SQLite in WebAssembly with full-text search
- `zod` — schema validation
- `dotenv` — env file loading (checks `.env.local` and `.env` in both cwd and data dir)

### Environment Variables

Chrome sync: `FT_CHROME_USER_DATA_DIR`, `FT_CHROME_PROFILE_DIRECTORY`
OAuth sync: `X_CLIENT_ID`, `X_CLIENT_SECRET`, `X_CALLBACK_URL`
Data location override: `FT_DATA_DIR`
