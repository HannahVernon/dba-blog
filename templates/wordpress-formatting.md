# WordPress Formatting Reference

Technical notes for publishing to sqlserverscience.com. These are hard-won lessons from actual publishing failures.

## Theme: Ascetica 0.3.5

- Base font: Georgia (overridden site-wide with **Droid Sans** via Google Fonts)
- Logo: `https://www.sqlserverscience.com/wp-content/uploads/2018/07/Capture-9.png` (454×111px)
- Logo dominant color: `#253055` (deep navy blue)
- `.entry-title` is very light/faded by default — needs CSS override if customizing

## wpautop Behavior

WordPress's `wpautop` filter automatically wraps text in `<p>` tags. This means:

- **Do NOT** wrap text in `<p>` tags yourself — you'll get double-wrapped paragraphs
- **DO** use double newlines between paragraphs — wpautop converts these to `<p>` breaks
- **DO** put blank lines around all block elements:
  - `<h2>`, `<h3>`, etc.
  - `<pre>` (code blocks)
  - `<ol>`, `<ul>` (lists)
  - `<blockquote>`
  - `<table>`
  - `<img>`
- Without blank lines, wpautop merges adjacent elements into a single `<p>` block

### Image-specific quirk

`<img>` tags **must** have blank lines above and below to get their own `<p>` block. Without blank lines, images get merged into adjacent text paragraphs and may not display correctly.

## Syntax Highlighting: Urvanov/Crayon

The site uses the Urvanov Syntax Highlighter (Crayon fork). Code blocks use `<pre>` tags with class attributes:

```html
<!-- T-SQL -->
<pre class="lang:tsql decode:true">SELECT 1;</pre>

<!-- PowerShell -->
<pre class="lang:ps decode:true">Get-Process | Sort-Object CPU -Descending</pre>

<!-- Shell/bash -->
<pre class="lang:sh decode:true">grep -r "pattern" /var/log/</pre>

<!-- XML/HTML -->
<pre class="lang:xhtml decode:true"><configuration><appSettings /></configuration></pre>

<!-- Plain text / command output -->
<pre class="lang:default decode:true highlight:0">Msg 207, Level 16, State 1
Invalid column name 'foo'.</pre>
```

**Notes:**
- `decode:true` is required — without it, HTML entities inside the block won't render correctly
- `highlight:0` on `lang:default` prevents line highlighting on plain text output
- Multi-line code blocks work fine inside a single `<pre>` tag
- Do NOT use markdown fenced code blocks — they won't render with syntax highlighting

## REST API Notes

### Base URL
```
https://www.sqlserverscience.com/wp-json/wp/v2
```

### Authentication
Basic auth with application password. Include these headers:
```
User-Agent: CopilotCLI/HannahDBABlog
Referer: https://copilot-cli.hannah-dba-blog/
```

### Reading post content
Always use `?context=edit` to get `.content.raw`. Without it, `.content.raw` is `null` — and if you accidentally update a post with that null value, it **wipes the entire post**.

### Creating/updating posts
- **JSON body** preserves newlines better and is preferred
- **Form-encoded body** is the fallback if JSON produces empty content
- After any update, **read back the post** and verify content length > 0

### Category and Tag IDs (current)

**Categories:**
internals(2), tools(3), configuration(13), security(19), basics(20), performance(27), sql-server-agent(35), t-sql(41), troubleshooting(53), reporting(82), ai-for-dbas(159), professional-development(188), hot-takes(194)

**Tags:**
automation(173), lock-escalation(178), indexing(179), memory-grants(180), concurrency(181), session-management(182), productivity(183), communication(189), soft-skills(190), teamwork(191), devops(192), database-seeding(193), azure-devops(195), naming(196), opinion(197), microsoft(198), sqlpackage(199), ssdt(200), synonyms(201), deployment(202)

## Publishing Schedule

- **Weekdays only** at **9:30 AM CDT** (14:30 UTC)
- One post per day, skip weekends
- Schedule each new post for the next available slot after the last scheduled post

## Yoast SEO

- Meta descriptions must be entered manually in the WordPress editor — the REST API does not support Yoast fields
- Keep meta descriptions under 155 characters
- Make them compelling and specific — not just a summary of the title
