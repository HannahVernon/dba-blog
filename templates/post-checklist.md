# Post Publishing Checklist

Use this checklist before and after publishing any post to sqlserverscience.com.

## Before Upload

- [ ] Content reviewed for real infrastructure names - anonymize all servers, databases, schemas, IPs, SPNs
- [ ] Gists and linked external content also sanitized (easy to miss)
- [ ] Code blocks use correct Crayon/Urvanov syntax highlighter classes:
  - T-SQL: `<pre class="lang:tsql decode:true">`
  - PowerShell: `<pre class="lang:ps decode:true">`
  - Shell/bash: `<pre class="lang:sh decode:true">`
  - XML/HTML: `<pre class="lang:xhtml decode:true">`
  - Plain text/output: `<pre class="lang:default decode:true highlight:0">`
- [ ] Blank lines surround all block elements (`<h2>`, `<pre>`, `<blockquote>`, `<table>`, `<ol>`, `<ul>`)
- [ ] No `<p>` tags in content - WordPress `wpautop` handles paragraph wrapping
- [ ] Double newlines between paragraphs
- [ ] `<img>` tags have blank lines above and below (wpautop requirement)
- [ ] Social closing links included (Bluesky + LinkedIn)
- [ ] Links to related posts where relevant (but don't force it; thin overlap is worse than no link)
- [ ] HTML entities in inline `<code>` tags use single encoding (`&lt;` not `&amp;lt;`)
- [ ] No editorial judgments about AI image generation (avoid "and you should" style endorsements)

## Upload Process

- [ ] Upload using **form-encoded body** for content (JSON can cause empty content - see note below)
  - **Exception:** JSON works fine if the content does NOT contain CDATA wrappers or other XML artifacts
  - JSON preserves newlines better than form-encoded - prefer JSON when possible, but verify content is non-empty after upload
- [ ] **Verify content is non-empty after upload** - read back with `?context=edit` and check `.content.raw` length
- [ ] Schedule for next available **weekday** - randomize time between **9:00-9:30 AM CDT** (14:00-14:30 UTC)
- [ ] Set category and tags (create new tags if needed)

## Featured Image

- [ ] Generate image using the [image prompt template](image-prompt.md)
- [ ] Check for Gemini watermark in bottom-right corner (sparkle icon)
  - If background is uniform/simple: remove with row-by-row color sampling (see Watermark Removal below)
  - If background is complex/textured: skip removal if watermark is nearly invisible against the background
- [ ] Convert PNG to WebP at quality 90 using `cwebp.exe`
- [ ] Upload WebP to WordPress media library
- [ ] Set **alt text on the media item** (not just the inline `<img>` tag)
- [ ] Set as featured image on the post (`featured_media`)
- [ ] Embed inline at top of post content with descriptive `alt` attribute
- [ ] Use the **medium_large** (768px) size for inline display - not the full resolution
- [ ] No `<a>` wrapper - images are illustrative, not click-to-zoom
- [ ] Verify image is on its own paragraph (blank line after `<img>` tag, or wpautop wraps it with text)

## Yoast SEO (Automated via mu-plugin)

The mu-plugin `wp-content/mu-plugins/expose-yoast-rest.php` exposes Yoast meta fields to the REST API.

- [ ] Set focus keyphrase and meta description via REST API:
  ```
  POST /wp-json/wp/v2/posts/{id}
  { "meta": { "_yoast_wpseo_focuskw": "...", "_yoast_wpseo_metadesc": "..." } }
  ```
- [ ] Meta description must be **under 155 characters** (orange bar = too long)
- [ ] Open the post once in the WordPress editor to trigger Yoast's JS scoring (traffic lights stay grey until then)
- [ ] Verify green/orange SEO and Readability scores after opening in editor

## After Upload

- [ ] Preview the post (visit `?p={post_id}` while logged in)
- [ ] Verify code blocks render with syntax highlighting
- [ ] Verify lists render as actual lists (not inline text)
- [ ] Verify image displays correctly (no text wrapping around it)
- [ ] Verify scheduled date/time is correct
- [ ] Check for double-encoded HTML entities (`&amp;lt;` rendering as literal `&lt;` text)

## Known Gotchas

### Content disappears after update
If you read a post without `?context=edit`, the `.content.raw` field is `null`.  If you then update the post using that null value, **it wipes the entire post content**.  Always use `?context=edit` and verify content is non-empty before updating.

### Double-encoded HTML entities
When content passes through JSON encoding and then form-encoded updates, angle brackets in `<code>` tags can get double-encoded: `&lt;img&gt;` becomes `&amp;lt;img&amp;gt;`, which renders as literal text instead of the intended `<img>`.  After uploading, search the raw content for `&amp;lt;` and `&amp;gt;` and fix any occurrences.

### Lists render inline
`wpautop` needs blank lines between `<li>` items and between the list and surrounding content.  If a list renders as a run-on paragraph, add blank lines around the `<ol>`/`<ul>` tags.

### CDATA wrappers
Never include `<![CDATA[...]]>` in content files.  This is an XML escaping mechanism that WordPress doesn't understand and can cause JSON uploads to produce empty content.

### Featured images don't auto-display
The Ascetica theme does NOT automatically display featured images in post content.  You must embed the image inline with an `<img>` tag in addition to setting `featured_media`.

### Image text wrapping
If the inline `<img>` tag is not separated from the following text by a blank line, wpautop may merge them into one `<p>` block, causing text to wrap alongside the image.  Always ensure a paragraph break after the image.

### Form-encoded updates strip newlines
When using `application/x-www-form-urlencoded` to update content, explicit `\r\n` sequences may get stripped.  The rendered HTML (`wpautop`) usually handles paragraph breaks correctly regardless, but verify the preview.

### Gist updates via `gh gist edit --add`
The `--add` flag on `gh gist edit` **appends** a new file rather than replacing an existing one with the same name.  To update gist content, delete and recreate the gist.  Remember to update any links in blog posts to the new gist ID.

## Watermark Removal

Gemini-generated images have a small sparkle watermark in the bottom-right corner.

**Detection:** Scan the bottom-right 150x150 pixel region for clusters of bright pixels (R/G/B > 200 for light backgrounds, or > 50 for dark backgrounds).

**Removal approaches:**
1. **Uniform background:** Fill the watermark bounds + margin with the sampled background color using `[System.Drawing.Graphics]::FillRectangle()`
2. **Gradient/varied background:** Use row-by-row color sampling from just left of the watermark, painting each row with its matching background color
3. **Complex/textured background:** If the watermark is nearly invisible against the background, skip removal entirely - a bad fill looks worse than a faint watermark

**Tools:** .NET `System.Drawing` (no Python dependency needed):
- `[System.Drawing.Bitmap]` for pixel access
- `[System.Drawing.Graphics]::FromImage()` for drawing
- `[System.Drawing.SolidBrush]` for fill color

## WebP Conversion Pipeline

- **Tool:** `C:\temp\libwebp\libwebp-1.5.0-windows-x64\bin\cwebp.exe`
- **Quality:** 90 (Hannah's preferred setting)
- **Typical savings:** 88-94% vs PNG
- **Upload:** All new WebPs go to `/wp-content/uploads/YYYY/MM/`
- **MIME type:** Already configured in IIS `web.config` `<staticContent>` section
