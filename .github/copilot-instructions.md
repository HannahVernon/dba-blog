# Copilot Instructions for dba-blog

This repository manages blog content and publishing for [sqlserverscience.com](https://www.sqlserverscience.com/).

## Before Any Blog Work

Read all files in `templates/` before creating, uploading, or modifying blog posts:

- `templates/post-checklist.md` - full pre/post-publish checklist
- `templates/wordpress-formatting.md` - REST API details, formatting rules, known gotchas
- `templates/image-prompt.md` - brand-consistent image generation template

Do not rely on memory from prior sessions.  These files are the source of truth.

## WordPress REST API Constants

Property | Value
---------|------
Base URL | `https://www.sqlserverscience.com/wp-json/wp/v2`
Username | See `wp-env.local` (not committed; create from `wp-env.local.example`)
Auth | Basic auth with application password (ask Hannah each session)
User-Agent | `CopilotCLI/HannahDBABlog`
Referer | `https://copilot-cli.hannah-dba-blog/`
Read posts | Always append `?context=edit` to get `.content.raw`
WebP tool | `C:\temp\libwebp\libwebp-1.5.0-windows-x64\bin\cwebp.exe` (quality 90)

**The `date` field in the REST API is interpreted as local time (CDT), not UTC.**  To schedule a post for 9:15 AM CDT, pass `"date": "2026-06-07T09:15:00"`.  Do not pass UTC values in the `date` field; they will be interpreted as CDT and the post will publish at the wrong time.  The target scheduling window is **09:00-09:30 CDT** (randomized), **every day of the week**.

## Mandatory Post Upload Sequence

Every blog post upload must follow this sequence.  Do not skip steps.

1. **Draft the post** in Markdown in `drafts/`.  Sanitize all real infrastructure names.
2. **Convert to HTML** using Urvanov/Crayon `<pre>` tags for code blocks (not Markdown fences).
3. **Create the post** via REST API as `status: draft`.  Verify content is non-empty by reading back with `?context=edit`.
4. **Set category and tags.**  Create new tags if needed.  Never leave a post uncategorized.
5. **Set Yoast SEO** via the `meta` field:
   - `_yoast_wpseo_focuskw` - focus keyphrase
   - `_yoast_wpseo_metadesc` - meta description (under 155 chars)
   - `_yoast_wpseo_title` - SEO title (under 60 chars for Google SERP).  Set this when the post title exceeds 60 characters so Yoast does not flag it as too long.  The SEO title appears in search results and browser tabs; the post title appears on the page itself.
6. **Generate a featured image prompt** using `templates/image-prompt.md`.  Present it to Hannah.
7. **After Hannah provides the image:** convert PNG to WebP, upload to media library, set alt text on the media item, set `featured_media` on the post, and embed the image inline in the post body using the `medium_large` size (768px).
8. **Schedule the post** for the next available day, randomized between 09:00-09:30 CDT.  Check existing scheduled posts first.
9. **Verify** by reading back the post with `?context=edit`: confirm content length > 0, featured_media is set, category is assigned, Yoast fields are populated.

## Featured Image Brand Rules

All featured images must follow these brand anchors (details in `templates/image-prompt.md`):

- **Primary color:** `#253055` (deep navy blue from the site logo)
- **Format:** 1200x628px wide-format
- **Style:** Clean, modern, slightly whimsical
- **No text overlays** in the image
- **Images featuring people should prominently feature women** (women in STEM emphasis)
- Use visual metaphors, not literal screenshots

## Continuous Improvement

When you make a mistake that required Hannah to correct you, suggest running `/chronicle improve` so the correction gets captured in this file and is not repeated in future sessions.

## Known Pitfalls (Learned the Hard Way)

- **Content wipe:** Reading a post without `?context=edit` returns `.content.raw` as `null`.  Updating with that null value erases the entire post.  Always use `?context=edit`.
- **Featured image not visible:** The Ascetica theme does NOT auto-display featured images.  You must embed an `<img>` tag inline in the post body in addition to setting `featured_media`.
- **Wrong schedule time:** The `date` field is local CDT.  Passing `14:15` schedules for 2:15 PM CDT, not 9:15 AM.
- **IIS caching:** WordPress REST API updates sometimes don't persist due to IIS output caching.  If an update silently reverts, Hannah may need to restart IIS.
