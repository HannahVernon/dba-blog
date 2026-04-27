# dba-blog

Working files and publishing infrastructure for [sqlserverscience.com](https://www.sqlserverscience.com/).

## Structure

- `drafts/` — Markdown source for blog posts (including the 18-post **ALTER DBA ADD AGENT** series and satellite posts)
- `media/` — Featured images and visual assets
- `process/` — Editorial process logs and decision records
- `templates/` — Reusable publishing assets:
  - [`image-prompt.md`](templates/image-prompt.md) — Brand-consistent image generation prompt template
  - [`post-checklist.md`](templates/post-checklist.md) — Pre/post-publish checklist
  - [`wordpress-formatting.md`](templates/wordpress-formatting.md) — WordPress formatting reference, REST API notes, and known gotchas
- `plan.md` — Series plan, post index, and cross-reference maps

## Blog Series

The **ALTER DBA ADD AGENT** series helps traditional SQL Server DBAs understand how AI coding assistants (GitHub Copilot CLI, Claude Code) can help with day-to-day tasks. The series landing page is at [sqlserverscience.com/ai-for-dbas/alter-dba-add-agent/](https://www.sqlserverscience.com/ai-for-dbas/alter-dba-add-agent/).

## Publishing

Posts are published to WordPress via the REST API. See [`templates/post-checklist.md`](templates/post-checklist.md) for the full workflow and [`templates/wordpress-formatting.md`](templates/wordpress-formatting.md) for technical details.
