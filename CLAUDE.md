# Project Instructions

## Session logs

When the user asks for a session summary at the end of a session, save it as a markdown file in `_session_logs/` using the filename format `YYYY-MM-DD-short-slug.md` (use the current date plus a short kebab-case slug derived from the one-line session title — e.g., `2026-05-25-rebuilt-broken-al-folio.md`). The slug should be 3–6 words max.

If the `_session_logs/` folder does not exist, create it. The folder is listed in `.gitignore`, so files inside it are local-only and never committed.

Use this exact structure:

```markdown
# Session — YYYY-MM-DD

> _Very short one-line title that captures the gist or main accomplishment of the session._

## What was done
(numbered list)

## Decisions made
(what was chosen and why)

## Blog post ideas
(anything interesting that came up)

## Next steps
(what's left to do)
```

The one-line title goes directly under the H1 as a blockquote. Keep it concise (one sentence, ideally under 12 words) — it should read like a headline summarizing the session at a glance.

If a file for today's date already exists, append a new dated heading inside it rather than overwriting.
