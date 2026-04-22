# Process Log — ALTER DBA ADD AGENT

This file captures the editorial process behind the blog series, in chronological order.
It exists so that Post 16 ("How This Series Was Written") has raw material to draw from.

---

## Session 1 — 2026-04-21

### Phase 1: Style Analysis

The author asked the AI (GitHub Copilot CLI, powered by Claude Opus 4.6) to familiarize itself with the blog at sqlserverscience.com before planning the series.

**Pages analyzed:**
- Homepage (post listing)
- "Inspecting SQL Server TLS Certificates Without Credentials" (full post)
- "Can GitHub Copilot Optimize Your T-SQL? I Put It to the Test" (full post)
- About page

**Style observations identified:**
- Practitioner-to-practitioner tone — opens with relatable pain points ("If you've ever needed to...")
- First-person narrative with real-world context — actual use cases, not textbook examples
- Technical depth — full command output, actual numbers, real T-SQL
- Problem-first framing — tool/technique always in service of a concrete problem
- Already writing about Copilot CLI — the T-SQL optimization series and coding standards post exist

### Phase 2: Requirements Gathering

The AI asked 8 clarifying questions before producing any plan. Key decisions made:

| Question | Decision |
|---|---|
| Series length | Open-ended, 8-12+ posts, add over time |
| Tool focus | Primarily Copilot CLI, mention Claude Code where relevant (same Claude models power both) |
| Target audience | Start with DBAs who've never used an AI coding agent; assume ChatGPT familiarity and code-completion experience (SQL Prompt, IntelliSense) |
| Example format | Concrete examples with actual prompts and results, genericized to prevent exploitation |
| Opening approach | Dedicated "wake-up call" post about disruption before practical content |
| Structure | Themed groupings (Foundations, Daily Tasks, Troubleshooting, Advanced) |
| Series name | "ALTER DBA ADD AGENT: Practical AI for Database Professionals" — chosen from ~10 options; the SQL-flavored name won over generic AI marketing phrases |
| Deliverable | Series outline first, not actual post drafts |

### Phase 3: Initial Plan (v1)

14 posts across 5 themes produced. Author reviewed and provided additional guidance:

- Keep posts concise — DBAs are busy, no tomes
- Focus on practical/actionable advice; social/professional impact should serve the practical content
- "Try This Yourself" sections woven naturally, not a forced structure
- Write in post order — themes build on each other
- Cross-reference existing blog posts and other posts in the series throughout
- Add a post about Erik Darling's PerformanceMonitor and PerformanceStudio tools

### Phase 4: Plan Update (v2)

- Added Post 12: "AI-Native Monitoring: PerformanceMonitor, PerformanceStudio, and the MCP Revolution"
- Old Post 12 (custom monitoring) became Post 13, now positioned as complement to the new Post 12
- Renumbered Posts 13-14 to 14-15
- Updated cross-reference maps with external references
- Series now 15 posts

### Phase 5: Meta Post and Workspace Setup

Author requested all working content move to `C:\temp\dba-blog\` — envisioning this directory as the artifact for a final meta post that "comes clean" about the AI-assisted writing process.

- Added Post 16: "How This Series Was Written: A DBA and an AI Walk Into a Terminal"
- Created `process/` directory for conversation logs and editorial notes
- Created `drafts/` directory for post drafts
- Series now 16 posts

### Directory Structure

```
C:\temp\dba-blog\
  plan.md              — Series outline and editorial guidelines
  process/
    log.md             — This file (chronological process log)
    decisions.md       — Key editorial decisions and rationale
  drafts/              — Post drafts (empty until writing begins)
```

---

### Phase 6: Post 1 Drafting

**First draft** produced at ~850 words. Covered:
- Code completion vs coding agents distinction
- Concrete examples from real usage
- Junior vs senior DBA leverage shift
- Series setup

**Rubber-duck critique** identified these issues:
1. **Tone too "LinkedIn provocative"** — phrases like "blind spot that's going to cost you" and "DBAs who use AI are going to replace DBAs who don't" read more like AI evangelism than practitioner writing
2. **Slightly too long** — recommended trimming to 700-900 words, cutting bullet list from 5 to 3 examples
3. **Junior vs senior framing too adversarial** — reframe around leverage, not replacement
4. **Unsupported broad claims** — "80% of DBA work," "unlimited supply," "it's already happening" need softening or anchoring to personal experience
5. **Series setup section drifting into table of contents** — compress to one paragraph

**Revisions applied:**
- Lowered the temperature throughout — removed "going to cost you," replaced "part nobody wants to hear" with "the leverage shift"
- Cut from 5 example bullets to 3 strongest
- Reframed junior/senior as leverage difference, not competition
- Anchored claims to "in my experience" and "what I'm already seeing"
- Compressed series setup from 6 bullets to one paragraph
- Final version: ~750 words

### Phase 7: Post 2 Drafting

**First draft** produced at ~1,200 words. 11 sections covering all DBA task areas.

**Rubber-duck critique** identified:
1. **Slightly too long** for a survey post — trim ~15-20%
2. **Troubleshooting and Incident Response** sections overlap — merge them
3. **Learning Angle and Team Multiplier** sections feel like add-ons — merge into one "Beyond Task Execution" section
4. **Too many cross-references** in body — limit to 1-2 embedded links, lighter touch
5. **Closing too generic** — tighten the bridge to Post 3

**Revisions applied:**
- Merged Troubleshooting + Incident Response into one section
- Merged Learning Angle + Team Multiplier into "Beyond Task Execution"
- Reduced from 11 sections to 9
- Lightened cross-references: removed sql-cert-inspector and ai-security-audit inline links, kept PerformanceMonitor as the one tool mention (it naturally fits the monitoring section)
- Kept existing blog post links in T-SQL section (they're earned — it's the author's own published work)
- Tightened closing
- Final version: ~900 words

### Phase 8: Post 3 Drafting

**First draft** produced at ~1,100 words. Covered installation, first interaction, context model, custom instructions, common mistakes, and a quick-win exercise.

**Rubber-duck critique** identified:
1. **Installation commands likely outdated** — old `@githubnext/github-copilot-cli` package name and `ghcs` command; should lead with WinGet on Windows
2. **Context section overstated** — "can see files in your directory" too absolute; need to mention trust prompts and explicit file references
3. **Custom instructions claims overpromised** — "every piece of T-SQL follows your rules" too absolute; soften to "usually gets much closer"
4. **Missing gotchas** — starting in wrong directory, being surprised by permission prompts
5. **GithubMarkdownViewer mention slightly forced** — deferred to generic "preferred markdown viewer"
6. **Tone slightly salesy** in two spots — "moment that clicks" and "read your code and understood it"

**Revisions applied:**
- Led with WinGet install, npm as alternate
- Added prominent disclaimer that CLI commands evolve — check official docs
- Rewrote context section around trusted folders and explicit file references
- Softened custom instructions claims
- Added two new gotchas (wrong directory, permission prompts)
- Toned down salesy lines
- Removed specific `~/.copilot/copilot-instructions.md` path (may change); linked to existing blog post for details
- Removed forced GithubMarkdownViewer mention
- Final version: ~1,050 words (slightly longer than posts 1-2, appropriate for a setup/tutorial post)

---

## Session 2 — 2026-04-22 (continued)

### Posts 4-8: Daily Tasks and Troubleshooting

Posts 4-8 were drafted, rubber-duck critiqued, and revised. Key catches:

- **Post 4:** Rubber-duck caught bad DMV example (per-database waits from session DMVs); replaced with Query Store approach
- **Post 5:** Caught oversimplified TLS example and deprecated Send-MailMessage; expanded sql-cert-inspector section
- **Post 6:** Standard critique, minor fixes
- **Post 7:** Standard critique; Hannah manually edited to add John Ness' SQL-Server-Scripts reference and validation paragraph
- **Post 8:** First post to receive **senior DBA persona critique** at Hannah's request. Caught incompatible lock modes (S blocking S), missing rollback cost warning, missing 1205 retry logic. Second round of fixes applied.

### Posts 9-16: Senior DBA Critique Standard

Starting Post 8, all posts received the senior DBA persona critique. Key patterns across all critiques:

**Recurring AI weaknesses caught:**
- Overclaiming what T-SQL DMVs can detect (AD account status, TLS version from DMVs)
- Clean scenarios that don't match messy production reality
- Absolute language ("every," "always," "complete") where "usually" or "most" is accurate
- Missing security/privacy considerations for data in AI prompts
- Glossing over implementation complexity (consecutive-check alerts need persisted state)
- sp_OACreate recommendation (removed — expands attack surface)
- pg-extract-schema reference in SQL Server post (replaced with SSDT/sqlpackage)

**Technical corrections by post:**
- Post 9: Query Store stores interval stats not per-execution; plan forcing affects future compilations not current query; 12-minute resolution claim too clean
- Post 10: Can't detect disabled AD accounts from SQL Server; encrypt_option shows connection encryption not Force Encryption config; PCI-DSS version not specified
- Post 11: DMA is being phased out; deprecated vs removed features conflated; AG rollback after failover is not simple; "rebuild statistics" is cargo-cult
- Post 12: Missing conflict disclosure; "nothing touches your server" too strong; MCP security boundaries not explained; too promotional
- Post 13: DMV join complexity glossed over; sp_OACreate inappropriate recommendation; HTML dashboard too toy-like without caveats
- Post 14: Instruction file paths incomplete; "it remembers" overstates persistence; NOLOCK stance too absolute; pg-extract-schema wrong tool
- Post 15: AI as teacher too idealized; code review overclaims static analysis; missing adoption friction and career honesty
- Post 16: Process log incomplete; quantification missing; "irony is the point" too cute; needs direct skepticism acknowledgment

**Estimated total corrections:** ~40-50 substantive technical, ~30+ tone/overclaim adjustments across 16 posts.

### Posts 1-7: Senior DBA Second-Round Critique (2026-04-22)

Posts 1-7 originally received only standard rubber-duck critiques. Hannah requested senior DBA persona reviews for these posts to match the standard applied to posts 8-16. All 7 reviews ran in parallel and returned significant findings.

**Key technical catches across posts 1-7:**

- **Post 1:** Overclaimed agent autonomy ("examines execution plans," "agent has all of those"); missing caution paragraph about agent limitations; "closes the knowledge gap" too broad
- **Post 2:** "Logic bugs the optimizer didn't catch" — wrong framing (optimizer doesn't catch logic bugs); security audits paragraph mashed unrelated concepts; migration examples too weak; missing limitations section
- **Post 3:** COUNT(1) over COUNT(*) is cargo-cult advice; COALESCE over ISNULL not interchangeable (different type precedence); "Say yes" to trust is reckless security advice; missing cloud data exposure warning
- **Post 4:** Query Store join path wrong — needs sys.query_store_plan between query and runtime_stats; blanket ISNULL→COALESCE replacement is dangerous; join-refactor claim too broad (only safe for inner joins); missing multi-row trigger handling caveat
- **Post 5:** AG + SIMPLE recovery check is nonsense (AG requires FULL); backup check too naive (no full/diff/log distinction, no CHECKSUM/test restore); AG "commit latency" not a clean DMV column; missing corruption/disk/error log checks
- **Post 6:** AG backup missing fn_hadr_backup_is_preferred_replica; offline index rebuild dangerous default for OLTP; SupportsShouldProcess doesn't magically validate -WhatIf behavior; missing script security checklist
- **Post 7:** "Show me what SQL it would actually execute" overstated; dependency mapping underestimates gaps (EXECUTE AS, CLR, Service Broker, encrypted modules); refactoring guidance too casual (locking, ordering, transaction scope changes); trigger section missing SQL Server-specific caveats

**All fixes applied in the same session.** Summary of changes:

| Post | Fixes Applied |
|---|---|
| 1 | Qualified autonomy claims, added "confidently wrong sometimes" caution, softened inevitability language, removed "closes the knowledge gap" |
| 2 | Fixed optimizer/logic-bug framing, split security paragraph, replaced migration examples, added verification caveats throughout, softened absolutes |
| 3 | Rewrote trust guidance with security-conscious approach, added cloud exposure warning, removed COUNT(1)/COALESCE as blanket advice, made custom instructions concrete |
| 4 | Fixed Query Store join path (added sys.query_store_plan), ISNULL→COALESCE now case-by-case caveat, outer join warning, MERGE pitfalls link, version/compat-level guidance |
| 5 | Replaced AG+SIMPLE nonsense with real AG health checks, added backup verification guidance (CHECKSUM, test restores), fixed commit latency to timestamp-derived estimates, added security/privacy warning |
| 6 | Added fn_hadr_backup_is_preferred_replica guidance, made offline rebuild opt-out not default, rewrote SupportsShouldProcess section with actual verification steps, added security checklist (credentials, TrustServerCertificate) |
| 7 | Fixed "actual SQL" to "likely rendered SQL," expanded dependency blind spots list, added refactoring risk checklist (locking, concurrency, transaction scope), trigger multi-row caveat, data governance warning |

All 16 posts have now received senior DBA persona critique and had fixes applied.

### Post 15 (New): AI-Assisted Pull Request Reviews for Database Code

Hannah requested a dedicated main-line post on using AI agents in PR review workflows. Previously only touched briefly in Posts 4 and 16.

**Drafted and critiqued in the same session.** Senior DBA critique caught:

- Semicolon/compat-level claim was factually wrong — semicolons already break specific constructs today
- "TRY/CATCH around DDL" presented as default best practice — naive; DDL error handling is operation-specific
- "Lock escalation risks without batching" — wrong vocabulary; the actual concern is Sch-M locks, size-of-data operations, log growth
- "BEGIN TRAN/COMMIT wrappers" treated as generic safety — can be actively dangerous (log growth, AG redo lag, long locks)
- "Effective permissions" overclaimed from script alone — requires live environment metadata
- "Approximate row counts" from SQL files — obvious overclaim
- "GRANT EXECUTE ON SCHEMA::dbo is a failure in most environments" — unsupported blanket statement
- Missing: HA/DR impact, transaction log/tempdb/version store, deployment tooling context, drop/recreate side effects, data migration/backfill scripts, type changes/nullability/collation, index review depth, dynamic SQL dependencies, SET options

**All fixes applied.** Post renumbered series: old Post 15 (DBA Team) → Post 16, old Post 16 (Meta) → Post 17. Cross-references updated in Posts 4, 14, 16, and 17. Series is now 17 posts.
