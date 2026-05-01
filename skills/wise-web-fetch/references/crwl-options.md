# crwl options reference

This file is loaded only when the `wise-web-fetch` skill body explicitly tells you to. It documents the parts of the `crwl` CLI relevant to the skill.

## Command shape

```bash
crwl crawl <url> [options]
```

## Output formats (`-o`, `--output`)

| Value | What it returns | Token cost vs WebFetch |
|-------|------------------|------------------------|
| `md-fit` | Markdown after a heuristic content filter that drops nav, footer, ads, comment widgets. **Default for this skill.** | 20-40% |
| `markdown` (alias `md`) | Full converted markdown, no filter. Lossless. | similar to WebFetch on size, but content can be richer because `crwl` runs JS while WebFetch does not |
| `all` | JSON with markdown plus the original HTML, links, media. Use only when explicitly debugging the page structure. | >100% |
| `json` | Structured-extraction output (requires `-j` or `-s`). Out of scope for v1. | n/a |

Both modes use Playwright under the hood, so JS-rendered SPAs return real content in either mode. WebFetch is HTTP-only and gets shell HTML on SPAs.

### Picking between `md-fit` and `markdown`

- `md-fit` is the default. Use it for summarization, Q&A, article reading, docs lookup, error-message lookup, debugging research — anything where the article body is the point.
- `markdown` (lossless) when the *links* or *images* are the point:
  - Citation extraction, link audit, source list, sitemap building.
  - Image-driven content (visual tutorial, photo article, gallery, screenshot-heavy guide). `md-fit` keeps most `[text](url)` links but drops some `![alt](url)` images, especially on pages that use SVG icons or CSS-background images instead of `<img>` tags.
  - Sparse-text pages (dashboards, table-only references) where `md-fit` is likely to over-filter.
  - User explicitly said "every link", "all references", "every image", "complete", "lossless".

If `md-fit` returns empty/under 100 chars on a page that should clearly have content, escalate to `markdown` (see `failure-handling.md`).

## Cache (`-bc`, `--bypass-cache`)

`crwl`'s `CACHE_MODE` defaults to `bypass` (no caching). The `-bc` flag is therefore unnecessary — every fetch is fresh by default.

## Verbose (`-v`)

Use only when the user asks for detailed crawl logs or you are diagnosing a failure. Adds noise to stdout.

## Other flags (out of scope for this skill)

The following options exist in `crwl` but are intentionally not used by `wise-web-fetch` v1: `-q` (LLM Q&A — needs API key), `-j`/`-s` (LLM extraction — same), `--deep-crawl` and `--max-pages` (multi-page traversal — Claude can iterate fetches itself), `-p` (browser profile — auth out of scope), `-B`/`-C`/`-f`/`-e` (config files — overkill).
