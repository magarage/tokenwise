# Failure handling reference

Loaded on demand by the `wise-web-fetch` skill. Catalog of error patterns and the response for each.

## Decision table

| Failure | How to detect | Action |
|---------|---------------|--------|
| `crwl` not on `PATH` | `command -v crwl` returns empty | Print install hint (pip and uv variants). Stop. Do NOT auto-fall-back to `WebFetch` — the user should install the dependency they opted into. |
| Playwright browser missing | stderr contains `Executable doesn't exist` or `playwright install` | Print `crawl4ai-setup` hint. Stop. Do NOT retry. |
| Empty `md-fit` output | exit 0, stdout empty or under 100 chars (mode was `md-fit`) | Escalate: re-run with `-o markdown` (lossless, still uses Playwright/JS rendering). If still under 100 chars, treat as failure and enter the retry path below. |
| Empty `markdown` output | exit 0, stdout empty or under 100 chars (mode was `markdown`) | Treat as failure. Enter the retry path below. No further escalation inside `crwl`. |
| Network timeout / DNS / unreachable / TLS error | non-zero exit, stderr does NOT mention `Executable doesn't exist` | Retry up to 3 attempts of `crwl crawl <url> -o <mode>`, with `sleep 3` between attempts. Empty-output safeguard applies after each attempt. |
| All retries exhausted | 3 retries failed | Invoke the built-in `WebFetch` tool with the same URL. Note: WebFetch does not run JavaScript, so SPAs may still return shell-only content. This is graceful-degradation last resort, not a quality fallback. |

## Detection snippets

### `crwl` missing

```bash
if ! command -v crwl >/dev/null; then
  echo "crwl not installed. Run: pip install -U crawl4ai && crawl4ai-setup (or uv tool install crawl4ai && crawl4ai-setup)"
  exit 1
fi
```

### Playwright browser missing

The error looks like:

```
Error: Executable doesn't exist at /Users/.../chromium-XXXX/chrome-mac/...
╔═════════════════════════════════════════════════════════════════════╗
║ Looks like Playwright was just installed or updated.                ║
║ Please run the following command to download new browsers:          ║
║                                                                     ║
║     playwright install                                              ║
║                                                                     ║
╚═════════════════════════════════════════════════════════════════════╝
```

Response: tell the user to run `crawl4ai-setup` (which wraps `playwright install` plus the crawl4ai-specific patches).

### Retry skeleton

```bash
url="$1"
mode="md-fit"   # or "markdown" if mode-selection picked lossless
for attempt in 1 2 3; do
  out=$(crwl crawl "$url" -o "$mode" 2>&1) && break
  sleep 3
done
```

## When to abort vs fall back

- Abort with hint (no fallback): missing `crwl`, missing Playwright browser. These are user-environment problems and silently switching to `WebFetch` would hide the install need.
- Escalate `md-fit` → `markdown` once: when md-fit returns empty output. Filter was too aggressive; lossless mode is still using Playwright so JS-rendered content is still captured.
- Fall back to `WebFetch`: only after 3 fetch attempts at runtime. Network glitches and dead URLs deserve graceful degradation. Be aware WebFetch is HTTP-only — for JS-rendered SPAs the fallback may itself return only shell HTML.
