# Featured Image Prompt Template

Use this template when generating featured images for sqlserverscience.com blog posts. The goal is visual brand consistency across all posts.

## Brand Anchors

- **Primary color:** `#253055` (deep navy blue — sampled from the site logo)
- **Accent palette:** Steel blue, white, with situational accent colors (red for errors/failures, green for success)
- **Style:** Clean, modern, slightly whimsical
- **Format:** Wide-format, 1200×628px (optimal for WordPress featured images and social sharing)
- **Text:** No text overlays in the image — the post title handles that
- **Representation:** Images featuring people should prominently feature women (women in STEM emphasis)

## Prompt Structure

```
Create a wide-format illustration (1200×628) for a technical blog post about [TOPIC].

[VISUAL METAPHOR — describe the scene, action, and key elements. Use metaphors
that map to the technical concept. Be specific about composition.]

Color palette: deep navy (#253055), steel blue, white, with [ACCENT COLOR]
accent for [PURPOSE]. Clean, modern, slightly whimsical style. No text overlays.
```

## Examples

### Database deployment problem
> Create a wide-format illustration (1200×628) for a technical blog post about SQL Server deployment problems with SqlPackage and synonyms.
>
> Show a conveyor belt / assembly line metaphor: on one end, neatly labeled boxes marked "DEV" enter the line. Midway through, a robotic arm (representing SqlPackage) is swapping address labels on packages — replacing "PROD" labels with "DEV" labels. At the end, a DBA (a woman with glasses) watches boxes tumble off the end of the belt, some with red X marks.
>
> Color palette: deep navy (#253055), steel blue, white, with red accent for the failures. Clean, modern, slightly whimsical style. No text overlays.

### Database seeding strategies
> Create a wide-format illustration (1200×628) for a technical blog post about database seeding — when to insert data directly vs. going through an API.
>
> Show a split scene: on the left, a person pours data (glowing cubes) directly into a database cylinder. On the right, the same data flows through a series of connected pipes and valves (representing an API layer) before reaching the database. Both paths lead to the same destination but the journey is visibly different.
>
> Color palette: deep navy (#253055), steel blue, white, with gold accent for the data cubes. Clean, modern, slightly whimsical style. No text overlays.

## Tips

- **Metaphors work better than literal screenshots** — a conveyor belt for deployment, pipes for data flow, a magnifying glass for troubleshooting
- **Keep it simple** — one central metaphor per image, not a busy collage
- **The navy blue should dominate** — it anchors the image to the site brand
- **Test at small sizes** — featured images render as thumbnails in post lists; make sure the composition reads at 300px wide
