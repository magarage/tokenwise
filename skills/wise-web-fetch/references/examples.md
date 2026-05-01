# Examples reference

Loaded on demand by the `wise-web-fetch` skill. Concrete usage patterns.

## Single URL

```bash
crwl crawl "https://example.com" -o md-fit
```

Returns filtered markdown to stdout. Pass through to your response context.

## Multiple URLs in one Bash call

```bash
for url in \
  "https://docs.python.org/3/library/asyncio.html" \
  "https://docs.python.org/3/library/concurrent.futures.html"; do
  echo "=== $url ==="
  crwl crawl "$url" -o md-fit
done
```

Use this shape when the user supplies a small list. For a large list, fetch sequentially and trim — Claude Code can iterate.

## Empty-output safeguard

```bash
url="https://example.com"
out=$(crwl crawl "$url" -o md-fit)
if [ "${#out}" -lt 100 ]; then
  out=$(crwl crawl "$url" -o markdown)
fi
echo "$out"
```

## Retry then fallback

Pseudocode (the skill body has the canonical version):

```text
for attempt in 1..3:
    run crwl crawl <url> -o md-fit
    if success and output >= 100 chars: return output
    if Playwright missing: print setup hint, stop
    sleep 3
# all retries failed:
call WebFetch tool with <url>
```

## Comparison vs WebFetch

| Page | WebFetch tokens (approx) | `crwl -o md-fit` tokens (approx) | `crwl -o markdown` tokens (approx) | Notes |
|------|--------------------------|----------------------------------|------------------------------------|-------|
| Python docs page (asyncio) | ~7,000 | ~2,000 | ~7,500 | Static page; md-fit saves ~70% |
| GitHub README (medium repo) | ~4,000 | ~1,500 | ~5,000 | md-fit saves ~60% |
| Blog post with sidebar/comments | ~9,000 | ~2,200 | ~9,500 | md-fit saves ~75% |
| TodoMVC React app (`/examples/react/dist/`) | ~50 (shell only — JS not executed) | ~30 (filter trims shell footer) | ~400 (Playwright renders the actual app) | SPA — WebFetch fails; lossless mode wins on content even though it costs more tokens |

Numbers are illustrative. md-fit saves tokens vs WebFetch on static pages; lossless markdown can use *more* tokens than WebFetch but is the only mode that returns useful content for JS-rendered SPAs. Pick mode per page.
