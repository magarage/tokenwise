---
name: wise-web-fetch
description: Use whenever you would otherwise call WebFetch to read web page content — for user requests OR your own autonomous research, documentation lookup, library docs, blog posts, external info gathering. Wise about web fetching — defaults to crawl4ai's md-fit mode (filters nav/footer/ads for 60-80% token savings vs WebFetch on static pages, the "tokenwise" path), and escalates to lossless Playwright-rendered markdown when the page is a JS-heavy SPA or every link/image must be preserved. The lossless mode can use MORE tokens than WebFetch but actually returns the rendered SPA content that WebFetch (HTTP-only) cannot see. DO NOT use when you need raw HTML/CSS/JS source, view-source, DOM structure, or to inspect page implementation — use curl or WebFetch for those.
---

# wise-web-fetch

Use the `crawl4ai` CLI (`crwl`) instead of the built-in `WebFetch` tool whenever you need a web page's *content*. The plugin is "wise" rather than always "tokenwise" — it picks the right mode for the page:

1. **Default — `md-fit` (the tokenwise path).** Heuristic content filter strips nav, footers, ads, and comment widgets before the markdown enters Claude's context — typically 60-80% fewer tokens than the lossless conversion `WebFetch` produces. Use this for ordinary content reads.
2. **Escalation — `markdown` (wise about CSR).** Lossless conversion of the page Playwright actually rendered. This can use *more* tokens than `WebFetch` because Playwright executes JavaScript and reveals content `WebFetch` (HTTP-only) never sees. Use it when every link/image matters, when the page is a JS-rendered SPA, or when `md-fit` over-filtered.

## When this skill applies

Apply this skill **before any `WebFetch` call** that would extract readable content. This includes:

- The user gave you a URL and asked you to read, summarize, or answer questions about it.
- You autonomously decided to look up a library's documentation, a blog post, an RFC, an error message lookup, or any other web reference while reasoning about a task.

## When this skill does NOT apply (use WebFetch instead)

Skip this skill and use the built-in `WebFetch` tool when the user (or the task) wants the page's **source** rather than its **content**:

- "Show me the raw HTML / CSS / JavaScript at <url>"
- "View source", "inspect the DOM", "check the markup"
- "How is this widget implemented in HTML?"
- The task involves reading `<script>`, `<style>`, or layout structure
- The user asks for a `view-source:` URL

If unclear, ask one clarifying question (content vs source) before deciding.

## Decision flow

1. **Intent gate.** Content or source? Source → abort skill, use `WebFetch` or `curl`.
2. **Availability check.** Run `command -v crwl`. Empty result → print the install hint below and stop. Do NOT fall back to WebFetch automatically — the user should install the dependency they asked for.
3. **Mode selection.** Default is `md-fit` (filtered). Pick `markdown` (lossless) only when the task needs comprehensive link or image preservation:
   - Cataloguing every link on the page (citation extraction, link audit, source list, "list all references", building a sitemap).
   - Image-driven content where `![alt](url)` markup matters (visual tutorial, photo article, image gallery, screenshot-heavy guide).
   - Sparse-text page where md-fit is likely to over-filter (dashboards, table-only references).
   - User explicitly said "every link", "all references", "every image", "complete", "lossless", or similar.

   For everything else — summarization, Q&A, article reading, docs lookup, error-message lookup, debugging research — use `md-fit`.

4. **Fetch.** Run `crwl crawl "<url>" -o <mode>` with the mode from step 3. For multiple URLs, loop in a single Bash call.
5. **Empty-output safeguard.** If exit was 0 but stdout is empty or under 100 characters AND mode was `md-fit`, escalate by re-running with `-o markdown`. If `markdown` is also empty/under 100 characters, treat as failure and proceed to step 7. (If you already ran `markdown` and it was empty, skip straight to step 7 — no further escalation inside `crwl`.)
6. **Setup error.** If stderr contains `Executable doesn't exist` or `playwright install`, print the setup hint below and stop. Do NOT retry.
7. **Retry.** On any non-zero exit other than the setup error, retry up to 3 attempts of `crwl crawl "<url>" -o <mode>`, with a fixed `sleep 3` between attempts. Apply the empty-output safeguard after each attempt.
8. **Fallback.** After 3 failed attempts, invoke the `WebFetch` tool with the same URL. (Note: WebFetch does not run JavaScript, so SPAs may still return shell-only content. This is a graceful-degradation last resort, not a quality fallback.)
9. **Output.** Pass `crwl`'s stdout through to your own response context as you would `WebFetch`'s output. No truncation.

## Install hints

When `crwl` is missing, tell the user:

> `crwl` (crawl4ai CLI) is not installed. Install with one of:
>
> ```bash
> pip install -U crawl4ai && crawl4ai-setup
> # or
> uv tool install crawl4ai && crawl4ai-setup
> ```

When the Playwright browser is missing, tell the user:

> The Playwright browser used by `crwl` is not installed. Run:
>
> ```bash
> crawl4ai-setup
> ```

## Reference material

For more detail (load on demand with the `Read` tool):

- `references/crwl-options.md` — full flag catalog and output-format comparison.
- `references/failure-handling.md` — error patterns and recovery decisions.
- `references/examples.md` — single-URL, multi-URL, and fallback examples.
