# Architecture Deep Dive — Field Theory CLI

Notes from onboarding walkthrough on 2026-04-05.

## How Bookmarks Are Pulled

Two sync methods, same output:

- **GraphQL (default, `ft sync`)** — Extracts `ct0` (CSRF) and `auth_token` cookies from Chrome's encrypted SQLite cookie DB via macOS Keychain. Calls X's internal GraphQL API (same as browser). Checkpoints every 25 pages, resumable. Stops when it catches up to stored bookmarks, hits a stale window, or reaches time/page limits. macOS only.

- **OAuth API (`ft sync --api`)** — Uses Twitter API v2 with OAuth 2.0 PKCE token. 100 bookmarks per page. Cross-platform but rate-limited. Requires developer credentials.

Both normalize tweets into `BookmarkRecord` objects and append to the JSONL cache.

### Chrome Cookie Extraction (GraphQL sync detail)

The GraphQL sync authenticates by extracting Chrome's session cookies. Here's the exact flow:

1. **Get decryption key from macOS Keychain** — Runs `security find-generic-password -w -s "Chrome Safe Storage" -a "Chrome"` to retrieve Chrome's encryption password. Tries multiple service/account name combos to support Chrome, Chromium, and Brave. Derives an AES key via `PBKDF2(password, 'saltysalt', 1003 iterations, 16 bytes, SHA1)`.

2. **Read Chrome's cookie SQLite DB** — Located at `~/Library/Application Support/Google/Chrome/{Profile}/Cookies`. Queries for cookies named `ct0` and `auth_token` on `.x.com` (falls back to `.twitter.com`). If Chrome has the DB locked (because it's open), copies the file to `/tmp` and queries the copy.

3. **Decrypt cookie values** — Chrome encrypts cookies with AES-128-CBC. The encrypted blob starts with `v10` (3 bytes), followed by ciphertext. IV is 16 space characters (`0x20`). On Chrome DB version 24+ (Chrome ~130+), the decrypted plaintext has a 32-byte SHA256 hash prepended that gets stripped.

4. **Build auth headers** — Returns `csrfToken` (the `ct0` value, used as `x-csrf-token` header) and `cookieHeader` (`ct0=...; auth_token=...`, sent as the `Cookie` header). These are the same credentials your browser uses when you browse x.com.

5. **Validation** — Checks that decrypted values are non-empty and contain only printable ASCII. If decryption fails (usually Chrome is open and the DB copy is stale, or wrong profile selected), provides specific troubleshooting steps.

## Data Storage: JSONL vs SQLite DB

**JSONL (`bookmarks.jsonl`) is the permanent source of truth.** It's an append-only, deduplicated log of raw bookmark data as received from the API.

**SQLite DB (`bookmarks.db`) is the derived query layer.** It's rebuildable from JSONL via `ft index`. It contains the same data plus classifications and full-text search indexes.

If you lose the DB → rebuild from JSONL and re-classify.
If you lose the JSONL → must re-sync from X.

## Index Rebuild Behavior

- **`ft index` (no force):** Keeps all existing rows and classifications. Loads existing IDs, skips them, inserts only new records from JSONL. FTS table gets a `rebuild` at the end to re-sync with the bookmarks table.

- **`ft index --force`:** Drops all tables (`bookmarks`, `bookmarks_fts`, `meta`) and recreates from scratch. All classifications are lost.

After sync, `ft index` runs automatically if new bookmarks were added.

## Classification

Two independent enrichment passes that write directly to the SQLite DB.

### Category vs Domain

- **Category** = the format/type of content (what kind of content it is): tool, security, technique, launch, research, opinion, commerce.
- **Domain** = the subject field (what topic it's about): ai, finance, defense, crypto, web-dev, devops, startups, health, politics, design, etc.

They're orthogonal. A Docker tutorial is category=technique, domain=devops. An AI agent framework is category=tool, domain=ai. An opinion about market cycles is category=opinion, domain=finance.

### How Classification Works

- **LLM-based (`ft classify`):** Shells out to `claude -p` or `codex exec` CLI. Batches of 50 bookmarks, ~2 min per batch. Sanitizes tweet text in `<tweet_text>` tags with prompt injection filters. Only processes unclassified bookmarks.

- **Regex-based (`ft classify --regex`):** Fast pattern matching, <1 second for entire corpus. Categories only, no domains.

### Dependency Order

Index must exist before classify can run. Classify reads from and writes to the DB via `UPDATE` statements — no index rebuild needed.

```
ft sync       → writes JSONL → auto-rebuilds index (new rows in DB)
ft classify   → reads unclassified rows from DB → writes categories/domains back
ft search     → reads from DB (works with or without classifications)
```

### `ft sync --classify` vs `ft classify`

Both do the same thing — run LLM classification on unclassified bookmarks. `--classify` is a convenience flag to avoid running two commands. Classification never happens automatically without the flag.

## DB Schema

**`bookmarks` table:** id, tweet_id, url, text, author info, timestamps, engagement metrics (likes/reposts/replies/quotes/bookmarks/views), media/link counts, links_json, tags_json, classification columns (categories, primary_category, domains, primary_domain), ingested_via.

**`bookmarks_fts` table (FTS5 virtual):** Indexes only `text`, `author_handle`, `author_name` for full-text search. Uses Porter stemming + unicode tokenizer. Content-synced to the bookmarks table.

**`meta` table:** Key-value, tracks `schema_version` (currently v3).

**B-tree indexes on:** `author_handle`, `posted_at`, `language`, `primary_category`, `primary_domain`.

### Why Categories Aren't in FTS

Category/domain filtering uses exact-match SQL `WHERE` clauses with B-tree indexes, not full-text search. FTS is for fuzzy text matching ("find bookmarks mentioning distributed systems"). Category filtering is exact ("show me all tool bookmarks"). You combine them via `ft search "query" --category tool` which hits FTS for text and WHERE for category.

## Scaling Considerations

At ~100 bookmarks/day:

- **JSONL:** ~1-2KB per bookmark. A year = ~36K bookmarks = ~50-70MB. Trivial to read line-by-line.
- **SQLite FTS5:** Handles millions of rows easily.
- **Index rebuild (`ft index`):** Only inserts new records, fast.
- **LLM classification:** The real bottleneck. 100 new/day = 2 batches = ~4 min. Skipping a week means ~28 min to catch up. Run `--classify` regularly to keep the backlog small.
- **`ft index --force`:** Avoid unless necessary — forces full LLM re-classification of the entire corpus.
