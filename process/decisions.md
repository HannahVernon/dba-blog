# Editorial Decisions — ALTER DBA ADD AGENT

Key decisions made during planning, with rationale. Referenced by the process log and useful for Post 16.

---

## Series Name

**Chosen:** "ALTER DBA ADD AGENT: Practical AI for Database Professionals"

**Rejected options:**
- "Terminal Velocity: A DBA's Guide to AI Coding Agents" — too generic
- "DBA Meets the Machine: Working with AI Coding Agents" — sounds like a movie tagline
- "The Command Line Copilot: AI Tools for Real DBAs" — too product-specific
- "Beyond Code Completion: AI Agents for Database Administrators" — accurate but boring
- "Pair Programming with a Robot: AI Agents for DBAs" — playful but misleading (it's not pair programming)
- "The DBA's New Toolkit: Coding Agents in the Real World" — forgettable
- "Teaching the Machine Your Job (Before Someone Else Does)" — too threatening
- "From SELECT * to SELECT Help: AI Coding Agents for DBAs" — cute but doesn't convey the scope

**Why ALTER DBA ADD AGENT won:** It speaks the audience's language (DDL syntax), it's memorable, and it captures the idea of augmentation — you're not replacing the DBA, you're adding a capability. The subtitle grounds it in practicality.

---

## Audience Starting Point

**Decision:** Assume experienced DBAs who have never used an AI coding agent, but have used ChatGPT and code-completion tools (IntelliSense, SQL Prompt).

**Rationale:** This is the largest underserved audience. DBAs who already use Copilot CLI don't need the Foundations theme. DBAs who've never heard of AI at all are probably not reading tech blogs. The sweet spot is "knows AI exists, maybe uses ChatGPT for ad-hoc questions, but hasn't connected an agent to their actual workflow."

---

## Tool Focus

**Decision:** Primarily GitHub Copilot CLI, mention Claude Code where relevant.

**Rationale:** The author's primary experience is with Copilot CLI using Claude models. Since both tools use the same underlying models (Claude Opus 4.6, etc.), most concepts transfer directly. Focusing on one tool avoids the "compare and contrast" pattern that makes posts longer without adding proportional value.

---

## Tone: No AI Marketing

**Decision:** Explicitly avoid AI hype language. No "revolutionary," "game-changing," "unleash the power of."

**Rationale:** The blog's existing tone is practitioner-to-practitioner. The audience (experienced DBAs) will tune out marketing language instantly. Show, don't tell — if the examples are compelling, the reader will draw their own conclusions about how transformative this is.

---

## Post Length

**Decision:** Keep posts concise and scannable. No tomes.

**Rationale:** DBAs are busy. A 10,000-word post won't get read. Each post should be a focused read that a DBA can get through during a lunch break or between incidents. If a topic is too big for one post, split it.

---

## "Try This Yourself"

**Decision:** Woven naturally into posts, not a mandatory section.

**Rationale:** Forced "exercises" feel like a textbook. Instead, when showing a technique, naturally suggest "try this with one of your own procedures" or "point it at your dev server and see what it finds." The reader should feel invited, not assigned homework.

---

## PerformanceMonitor / PerformanceStudio Post

**Decision:** Added as Post 12, positioned before the custom monitoring post (now Post 13).

**Rationale:** Erik Darling's tools are a perfect case study — they're free, open-source, built by a well-known SQL Server community member, and have AI integration (MCP servers) baked in from the start. This is concrete proof that the "AI-augmented DBA" isn't hypothetical. Placing it before the custom monitoring post creates a natural flow: "here's what purpose-built AI-integrated tools look like" → "here's how to build your own."

---

## Author's Tools

**Decision:** Link to the author's own open-source tools where they naturally fit a post's topic. Do not force them in — same principle as Erik Darling's tools.

**Tools identified and mapped:**
- **sql-cert-inspector** → Post 5 (TLS audit across a fleet)
- **SqlServerAgMonitor** → Posts 5, 6, 8 (AG monitoring, management, troubleshooting)
- **GithubMarkdownViewer** → Post 3 (viewing AI-generated markdown documentation)
- **ai-security-audit** → Post 10 (reusable AI prompts for security review)
- **pg-extract-schema / pg-deploy** → Post 11 (migration tooling, if PostgreSQL is relevant)
- **SQL-Server-Scripts** → Posts 4, 7 (real scripts to point the agent at)
- **MVCTSQLJobScripter** → Post 6 (scripting SQL Agent jobs)

**Rationale:** The author builds tools that solve real DBA problems. Mentioning them where relevant is authentic — the same way you'd mention any tool you actually use. The key constraint is that each mention must serve the post's instructional purpose, not the other way around.

---

## Meta Post (Post 16)

**Decision:** Final post reveals that the series was planned and written with AI assistance, using this directory as the artifact.

**Rationale:** It would be dishonest to write a series about AI-assisted DBA work without disclosing that the series itself was AI-assisted. More importantly, it's a powerful demonstration — the reader has already consumed 15 posts of useful content, and now learns the process behind it. The `dba-blog/` directory with all working files makes it reproducible and verifiable.
