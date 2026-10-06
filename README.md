# rdt-cli

[![PyPI version](https://img.shields.io/pypi/v/rdt-cli.svg)](https://pypi.org/project/rdt-cli/)
[![Python](https://img.shields.io/badge/python-%3E%3D3.10-blue.svg)](https://pypi.org/project/rdt-cli/)

> Fork von [jackwener/rdt-cli](https://github.com/jackwener/rdt-cli) (Upstream) — Änderung: `get_my_subscriptions` robust gegen doppelt-kodiertes JSON und 403 mit Cookie-Auth (siehe Commit-Historie).

A CLI for Reddit — browse feeds, read posts, search, and interact via reverse-engineered API 📖

## Features

- 🔐 **Auth** — auto-extract browser cookies, status check, whoami
- 🏠 **Feed** — browse home feed, popular, /r/all, and subscription-only feed (`--subs-only`)
- 📋 **Subreddits** — browse any subreddit with sort/time filters, view subreddit info
- 📰 **Posts** — read posts and comment trees with syntax highlighting
- 💬 **Expanded comments** — `--expand-more` loads additional `more comments` entries
- 🔢 **Short-index navigation** — `rdt show 3` to read, `rdt open 3` to browser
- 🔍 **Search** — full-text search with subreddit, sort, and time filters
- 📤 **Export** — export search results to CSV or JSON; `-o file.json` on any listing
- 👤 **Users** — view user profiles, post history, comment history, saved and upvoted items
- ⬆️ **Interactions** — upvote/downvote, save/unsave, subscribe/unsubscribe, comment (with 1.5-4s rate-limit delay)
- 🛡️ **Anti-detection** — consistent Chrome 133 fingerprint, `sec-ch-ua` alignment, Gaussian jitter, exponential backoff
- 📊 **Structured output** — `--yaml`, `--json`, `--output FILE`, `--compact`, `--full-text`
- 📦 **Stable envelope** — see [SCHEMA.md](./SCHEMA.md) for `ok/schema_version/data/error`
- 🤖 **Agent-friendly** — Rich output on stderr, `--compact` for token-efficient output

> **AI Agent Tip:** Prefer `--yaml` for structured output unless strict JSON is required. Non-TTY stdout defaults to YAML automatically. Use `--compact` to reduce token usage.

## Installation

```bash
# Recommended: uv tool (fast, isolated)
uv tool install rdt-cli

# Or: pipx
pipx install rdt-cli
```

Upgrade to the latest version:

```bash
uv tool upgrade rdt-cli
# Or: pipx upgrade rdt-cli
```

From source:

```bash
git clone git@github.com:eulrich101618/rdt-cli.git
cd rdt-cli
uv sync
```

## Usage

```bash
# ─── Auth ─────────────────────────────────────────
rdt login                             # Extract cookies from browser
rdt status                            # Check login status
rdt status --json                     # Structured JSON envelope
rdt whoami                            # Detailed profile (karma, account age)
rdt logout                            # Clear saved cookies

# ─── Browse ───────────────────────────────────────
rdt feed                              # Home feed (requires login)
rdt feed --subs-only                  # Subscriptions-only feed (no algorithm)
rdt feed --subs-only -n 5 --max-subs 10  # Limit per-sub posts and max subs
rdt popular                           # Popular posts
rdt popular --full-text               # Show full titles
rdt all                               # /r/all
rdt sub python                        # Browse subreddit
rdt sub programming -s top -t week    # Sort + time filter
rdt sub-info python                   # Subreddit info (subscribers, etc.)
rdt user spez                         # User profile
rdt user-posts spez                   # User's submitted posts
rdt user-comments spez                # User's comments
rdt saved                             # Your saved posts/items
rdt upvoted                           # Your upvoted posts

# Short index works after list commands (feed/popular/sub/search)
rdt sub python
rdt show 1                            # Read post #1 from listing
rdt open 1                            # Open post #1 in browser
rdt upvote 1                          # Upvote post #1

# ─── Reading ──────────────────────────────────────
rdt read 1abc123                      # Read post by ID
rdt read 1abc123 --expand-more        # Expand top-level "more comments"
rdt show 3                            # Read result #3 from last listing
rdt show 3 --expand-more              # Expand additional comments from cache-backed post
rdt show 1 -s top                     # Sort comments by top
rdt open 3                            # Open in browser

# ─── Search ───────────────────────────────────────
rdt search "python async"             # Global search
rdt search "rust vs go" -r programming  # Within subreddit
rdt search "ML" -s top -t year        # Sort by top, last year
rdt search "AI" -o results.json       # Save to file
rdt search "rust" --compact --json    # Compact agent output

# ─── Export ───────────────────────────────────────
rdt export "python tips" -n 100 -o tips.csv
rdt export "rust" --format json -o results.json

# ─── Interactions (require login) ─────────────────
rdt upvote 3                          # Upvote result #3
rdt upvote 3 --down                   # Downvote
rdt upvote 3 --undo                   # Remove vote
rdt save 3                            # Save result #3
rdt save 3 --undo                     # Unsave
rdt subscribe python                  # Subscribe to r/python
rdt subscribe python --undo           # Unsubscribe
rdt comment 3 "Great post!"           # Comment on result #3
```

## Authentication

rdt-cli supports browser cookie extraction to authenticate with Reddit:

1. **Saved cookies** — loads from `~/.config/rdt-cli/credential.json`
2. **Browser cookies** — auto-detects installed browsers and extracts cookies (supports Chrome, Firefox, Edge, Brave)

`rdt login` automatically tries all installed browsers and uses the first one with valid cookies.

### Cookie TTL

Saved cookies are valid for **7 days** by default. After that, the client automatically attempts to refresh from the browser. If browser extraction fails, the existing cookies are used with a warning.

### Short-Index Navigation

After any listing command such as `feed`, `popular`, `all`, `sub`, or `search`, the CLI stores the latest ordered post list in `~/.config/rdt-cli/index_cache.json`.

- `rdt show <N>` reads the Nth post from the latest listing
- `rdt open <N>` opens the Nth post in the browser
- `rdt upvote <N>`, `rdt save <N>`, `rdt comment <N>` reuse the same short index
- Empty listings clear the index cache, so old results are not reused by accident

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `OUTPUT` | `auto` | Output format: `json`, `yaml`, `rich`, or `auto` (→ YAML when non-TTY) |

## Rate Limiting & Anti-Detection

rdt-cli includes anti-detection measures designed to minimize risk:

### Request Timing
- **Gaussian jitter**: Delays between requests use a truncated Gaussian distribution (~1s mean, σ=0.3)
- **Random long pauses**: ~5% of requests include an additional 2-5 second delay simulating reading behavior
- **Auto-retry**: Exponential backoff on HTTP 429/5xx and network errors (up to 3 retries)

### Browser Fingerprint Consistency
- **UA/Platform alignment**: User-Agent, `sec-ch-ua`, `sec-ch-ua-platform`, `sec-ch-ua-mobile` are all consistent (Chrome 133)
- **Cookie merge**: Set-Cookie headers from Reddit responses are merged back into the session

## Structured Output

All `--json` / `--yaml` output uses the shared envelope from [SCHEMA.md](./SCHEMA.md):
```yaml
ok: true
schema_version: "1"
data: { ... }
```

When stdout is not a TTY (e.g., piped or invoked by an AI agent), output defaults to YAML.
Use `OUTPUT=yaml|json|rich|auto` to override.

## Use as AI Agent Skill

rdt-cli ships with a [`SKILL.md`](./SKILL.md) that teaches AI agents how to use it.

### [Skills CLI](https://github.com/vercel-labs/skills) (Recommended)

```bash
npx skills add eulrich101618/rdt-cli
```

| Flag | Description |
| --- | --- |
| `-g` | Install globally (user-level, shared across projects) |
| `-a claude-code` | Target a specific agent |
| `-y` | Non-interactive mode |

### Manual Install

```bash
mkdir -p .agents/skills
git clone git@github.com:eulrich101618/rdt-cli.git .agents/skills/rdt-cli
```

## Project Structure

```text
rdt_cli/
├── __init__.py           # Version
├── __main__.py           # python -m rdt_cli entry point
├── cli.py                # Click entry point & command registration
├── client.py             # Reddit API client (rate-limit, retry, anti-detection)
├── auth.py               # Cookie authentication + TTL refresh
├── constants.py          # URLs, headers, sort options
├── exceptions.py         # Error hierarchy (6 exception types)
├── index_cache.py        # Short-index cache for show/open commands
└── commands/
    ├── _common.py        # Shared helpers (envelope, output routing, formatters)
    ├── auth.py           # login, logout, status, whoami
    ├── browse.py         # feed, popular, all, sub, sub-info, user, user-posts, user-comments, saved, upvoted, open
    ├── post.py           # read, show
    ├── search.py         # search, export
    └── social.py         # upvote, save, subscribe, comment
```

## Development

```bash
# Install dependencies
uv sync

# Run tests
uv run pytest tests/ -v

# Unit tests only (no network)
uv run pytest tests/ -v -m "not smoke"

# Smoke tests (need cookies)
uv run pytest tests/ -v -m smoke

# Lint
uv run ruff check .
```

## Troubleshooting

**Q: `No Reddit cookies found`**

1. Open any browser and visit https://www.reddit.com/
2. Log in with your account
3. Run `rdt login` (auto-detects browser)

**Q: `database is locked`**

Close the browser Cookie database lock — close browser, then retry `rdt login`.

**Q: `Session expired`**

Your cookies have expired. Run `rdt logout && rdt login` to refresh.

**Q: `Rate limited`**

Wait and retry; the built-in exponential backoff handles this automatically.

**Q: Requests are slow**

The built-in Gaussian jitter delay (~1s between requests) is intentional to mimic natural browsing and avoid triggering Reddit's rate limiting.

---

## License

Apache-2.0
