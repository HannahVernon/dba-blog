# Post Publishing Checklist

Use this checklist before and after publishing any post to sqlserverscience.com.

## Before Upload

- [ ] Content reviewed for real infrastructure names — anonymize all servers, databases, schemas, IPs, SPNs
- [ ] Code blocks use correct Crayon/Urvanov syntax highlighter classes:
  - T-SQL: `<pre class="lang:tsql decode:true">`
  - PowerShell: `<pre class="lang:ps decode:true">`
  - Shell/bash: `<pre class="lang:sh decode:true">`
  - XML/HTML: `<pre class="lang:xhtml decode:true">`
  - Plain text/output: `<pre class="lang:default decode:true highlight:0">`
- [ ] Blank lines surround all block elements (`<h2>`, `<pre>`, `<blockquote>`, `<table>`, `<ol>`, `<ul>`)
- [ ] No `<p>` tags in content — WordPress `wpautop` handles paragraph wrapping
- [ ] Double newlines between paragraphs
- [ ] `<img>` tags have blank lines above and below (wpautop requirement)
- [ ] Social closing links included (Bluesky + LinkedIn)
- [ ] Links to related posts in the series where relevant

## Upload Process

- [ ] Upload using **form-encoded body** for content (JSON can cause empty content — see note below)
  - **Exception:** JSON works fine if the content does NOT contain CDATA wrappers or other XML artifacts
  - JSON preserves newlines better than form-encoded — prefer JSON when possible, but verify content is non-empty after upload
- [ ] **Verify content is non-empty after upload** — read back with `?context=edit` and check `.content.raw` length
- [ ] Schedule for next available **weekday at 9:30 AM CDT** (14:30 UTC)
- [ ] Set category and tags (create new tags if needed)

## Featured Image

- [ ] Generate image using the [image prompt template](image-prompt.md)
- [ ] Upload to WordPress media library
- [ ] Set **alt text on the media item** (not just the inline `<img>` tag)
- [ ] Set as featured image on the post (`featured_media`)
- [ ] Embed inline at top of post content with descriptive `alt` attribute

## After Upload

- [ ] Preview the post (visit `?p={post_id}` while logged in)
- [ ] Verify code blocks render with syntax highlighting
- [ ] Verify lists render as actual lists (not inline text)
- [ ] Verify image displays correctly
- [ ] Enter Yoast meta description manually (REST API doesn't support Yoast fields)
- [ ] Verify scheduled date/time is correct

## Known Gotchas

### Content disappears after update
If you read a post without `?context=edit`, the `.content.raw` field is `null`. If you then update the post using that null value, **it wipes the entire post content**. Always use `?context=edit` and verify content is non-empty before updating.

### Lists render inline
`wpautop` needs blank lines between `<li>` items and between the list and surrounding content. If a list renders as a run-on paragraph, add blank lines around the `<ol>`/`<ul>` tags.

### CDATA wrappers
Never include `<![CDATA[...]]>` in content files. This is an XML escaping mechanism that WordPress doesn't understand and can cause JSON uploads to produce empty content.

### Featured images don't auto-display
The Ascetica theme does NOT automatically display featured images in post content. You must embed the image inline with an `<img>` tag in addition to setting `featured_media`.
