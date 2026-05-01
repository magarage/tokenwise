# tokenwise
> Think tokenwise.  
> Be tokenwise.  
> Tokenwise your prompts.  

Token-efficient external tooling for Claude Code.

The first plugin in this repo is **`wise-web-fetch`**, a skill that replaces Claude Code's built-in `WebFetch` tool with the [crawl4ai](https://github.com/unclecode/crawl4ai) CLI. It is "wise" rather than always "tokenwise" — the skill picks the right mode for the page rather than always optimizing for fewest tokens.

## Why

Two distinct wins, used in different situations:

1. **`md-fit` mode (the tokenwise path).** Heuristic content filter strips nav, footers, ads, and comment widgets — typically 60–80% fewer tokens than the lossless markdown `WebFetch` returns on the same static page. Used for ordinary content reads.
2. **`markdown` mode (wise about CSR).** Lossless conversion of the page Playwright actually rendered. JS-heavy single-page apps (React/Vue/Angular/etc.) return their real content here — `WebFetch` is HTTP-only and only ever sees the empty shell HTML, so for an SPA `WebFetch` returns nothing useful. This mode can use *more* tokens than `WebFetch` because Playwright reveals content `WebFetch` never sees.

Tradeoff in plain language: on static pages we save tokens. On SPAs we sometimes spend more tokens than `WebFetch` would, but we actually get the content. Always picking md-fit isn't the right answer; always picking markdown isn't either. The skill picks per page.

## Install

### 1. Install `crawl4ai` (one-time)

Pick one:

```bash
# pip
pip install -U crawl4ai && crawl4ai-setup

# uv tool (isolated environment, recommended)
uv tool install crawl4ai && crawl4ai-setup
```

Verify:

```bash
crwl crawl https://example.com -o md-fit
```

You should see the example.com markdown.

### 2. Install the plugin in Claude Code

```text
/plugin marketplace add magarage/tokenwise
/plugin install tokenwise@tokenwise
```

The `wise-web-fetch` skill becomes available immediately. Claude will pick it over `WebFetch` whenever a URL needs reading.

## Usage

You usually do not invoke the skill directly. Just ask Claude to read a URL:

- "Summarize https://example.com"
- "Look up the Python `asyncio.gather` docs"
- "What does the changelog of <repo> say about v2?"

Claude detects the intent, runs `crwl`, and pipes the filtered markdown back into the conversation.

## When the skill skips itself

The skill is for **content**, not **source**. It deliberately steps aside for requests like:

- "Show me the raw HTML at <url>"
- "View source of this page"
- "What CSS rules style this element?"
- "How is this component implemented in markup?"
- `view-source:` URLs

For those, use `curl` (raw HTML) or `WebFetch` — `crwl` returns rendered markdown and isn't suited to source inspection.

## Failure behavior

| Situation | Behavior |
|-----------|----------|
| `crwl` not installed | Skill prints install hint and stops. Does NOT silently fall back to WebFetch. |
| Playwright browser missing | Skill prints `crawl4ai-setup` hint and stops. |
| `md-fit` returned empty (under 100 chars) | Skill escalates to `crwl -o markdown` (lossless, still JS-rendered). |
| Network failure | Skill retries up to 3× with a 3-second sleep between attempts. |
| All retries fail | Skill falls back to the built-in `WebFetch` tool. Note: WebFetch is HTTP-only, so SPAs may still return empty content — this is graceful degradation, not a quality fallback. |

## Limitations (v1)

- No authentication / browser profiles.
- No JavaScript interaction (clicks, form fills) — the page is rendered once with default state.
- No deep-crawl across multiple pages — Claude iterates fetches itself if needed.
- No LLM-backed Q&A or JSON extraction. These features exist in `crwl` (`-q`, `-j`) but require a separate LLM API key. Claude answers directly from the fetched markdown.

## License

Apache-2.0. See `LICENSE`.
