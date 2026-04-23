# How This Series Was Written: A DBA and an AI Walk Into a Terminal

If you've read this far in the series, you've consumed seventeen posts about how AI coding agents can transform DBA workflows. You may have already guessed the punchline.

This entire series was planned, outlined, drafted, and refined using the very AI coding agents it describes.

## The Setup

The tool was GitHub Copilot CLI, powered by Claude Opus 4.6 — the same tool recommended throughout the series. The working directory was `C:\temp\dba-blog\`, and every draft, every editorial decision, and every iteration was captured in files that still exist in that directory.

I didn't write these posts from scratch, and I didn't just ask the AI to "write a blog post about AI for DBAs." The process was collaborative in the way this series has been describing: I provided the direction, the judgment, and the verification. The AI provided speed, structure, and a tireless willingness to argue with itself.

Let me be direct about what "coauthored" means here: the AI generated first-draft prose from my outlines and key points. I wrote every prompt, chose every example topic, verified technical claims against documentation and my own production experience, and made manual edits where my judgment disagreed with the draft. Some posts needed light editing. Others needed significant rewriting — especially where operational reality didn't match the AI's clean scenarios.

## How It Started

I opened a terminal session and told the agent I wanted to write a blog series. Before it wrote a single word, it asked me eight questions:

- How many posts?
- What's the primary tool focus?
- Who's the target audience?
- What tone — technical, conversational, marketing?
- Should there be a "wake-up call" post or jump straight to practical content?
- What structure — themed groupings, difficulty progression, or random?
- What should examples look like — conceptual or real output?
- What's the deliverable — outline first or start writing?

Those questions shaped everything that followed. The AI didn't assume it knew what I wanted. It gathered requirements the way a good consultant would — except faster and without billing me by the hour.

## The Plan

The agent produced a 14-post outline across five themes: Foundations, Daily Tasks, Troubleshooting, Security, and Advanced Topics. I reviewed it, added a post about Erik Darling's PerformanceMonitor and PerformanceStudio tools, requested a meta post (this one), and asked for a satellite post pipeline — focused subtopics that could spin off into their own posts later.

The final plan: 18 core posts plus 79 satellite post candidates across 8 topic areas. The plan lived in `plan.md` and evolved throughout the writing process.

## The Writing Process

Each post followed this workflow:

1. **Draft** — I gave the agent the post topic, key points from the plan, and any specific examples or tools to include. The agent produced a first draft, usually 800-1,200 words.

2. **Senior DBA Critique** — Starting with Post 8, I had the agent adopt a persona: a skeptical 20-year DBA veteran who fact-checks every claim. This "senior DBA reviewer" was brutal — and caught real technical errors that would have embarrassed me in front of my audience.

3. **Revisions** — I applied the fixes the critique identified. Sometimes this meant rewriting entire sections. Sometimes it meant adding a single caveat.

4. **My Review** — I read every post, made manual edits where my experience disagreed with the agent's draft, and added details the agent couldn't know (specific tool references, personal anecdotes, community context).

5. **Publish** — After my review and any final adjustments.

## What the Senior DBA Reviewer Caught

The senior DBA critique persona was the most valuable part of the process. Here's a sample of what it caught:

**Post 8 (Wait Stats and Deadlocks):** The initial draft showed a blocking chain where an S lock was blocked by another S lock. S locks are compatible — they don't block each other. A senior DBA would have caught that in the first paragraph and stopped trusting the rest of the post.

**Post 9 (Incident Response):** The draft described querying "the last 5 executions from Query Store." Query Store doesn't store per-execution data — it stores aggregated interval-based statistics. The draft also claimed a 12-minute incident resolution, which the reviewer called "too Hollywood" for a blog targeting experienced DBAs.

**Post 10 (Security Audits):** The draft claimed you could detect disabled Active Directory accounts from SQL Server metadata. You can't — SQL Server knows about SQL login disablement, not AD account status. The draft also suggested checking TLS protocol versions from DMVs, when the DMVs actually show TDS protocol, not negotiated TLS version.

**Post 11 (Migration):** The draft oversimplified AG rolling upgrades, glossing over the fact that after you fail over to an upgraded replica, "rolling back" means restoring from backup and rebuilding the AG — not just failing back.

**Post 12 (PerformanceMonitor):** The reviewer pointed out the missing conflict-of-interest disclosure (Erik Darling is a friend) and flagged that the post read like vendor copy instead of a field-tested recommendation.

Every one of these would have damaged my credibility with exactly the audience I'm trying to reach. The critique surfaced the issues, but I verified each correction against SQL Server documentation on Microsoft Learn or my own production experience before applying it. Same-system self-review isn't independent QA — the corrections only count because a human with domain expertise validated them.

Across the full series, the senior DBA critique identified roughly 40-50 substantive technical corrections and another 30+ tone/overclaim adjustments. That's a meaningful error rate — not catastrophic, but high enough that publishing raw AI drafts without review would have been irresponsible.

**A note on Posts 1-7:** the senior DBA critique persona started with Post 8. Posts 1-7 initially received a standard rubber-duck review — less aggressive, and it missed issues that the harder review would have caught. I went back and ran all seven posts through the senior DBA critique, which surfaced significant problems: Post 3 recommended "say yes" to trust prompts (reckless), Post 4 had the wrong Query Store join path, Post 5 checked for AG databases in SIMPLE recovery mode (AG databases must be in FULL), and Post 6 defaulted to offline index rebuilds for OLTP. All were fixed before publication — but they illustrate exactly why the senior DBA persona was so valuable. The standard rubber-duck missed them; the skeptical 20-year veteran didn't.

## What the AI Got Wrong

Beyond the technical errors the reviewer caught, there were patterns in what the AI consistently struggled with:

**Operational reality.** The AI produces clean scenarios where diagnosis leads neatly to resolution. Real incidents have red herrings, multiple simultaneous symptoms, and political dimensions the AI can't model.

**Confidence calibration.** The AI's first drafts were too certain. "The agent generates a complete script" became "the agent generates a draft that you validate." "It remembers your environment" became "it loads your context — but doesn't enforce it." Every post needed the temperature turned down.

**Completeness claims.** The AI would describe a monitoring query as "comprehensive" when it was missing obvious checks. The senior DBA reviewer consistently added "what about X, Y, and Z?" — and was usually right.

**Security awareness.** Early drafts were cavalier about putting environment details in instruction files and pasting production data into AI conversations. Those needed explicit security guidance added.

**Residual blind spots.** Even after review, subtle mistakes may survive — especially around edge cases, old SQL Server versions, unusual configurations, and the operational nuance that only comes from running specific workloads in specific environments. The AI tends to overfit to mainstream documentation and default configurations. If something in this series doesn't match your experience, your experience is probably right.

## What the AI Got Right

**Structure.** The series outline, the themed groupings, the cross-reference map, the satellite post pipeline — all came from the AI on the first pass and survived mostly intact through the final version.

**Prompt examples.** The prompts shown in each post are realistic and well-structured. They demonstrate the kind of context-rich prompting that gets good results from AI agents.

**Tone matching.** After analyzing my existing blog posts, the agent matched my writing style closely — practitioner-to-practitioner, problem-first, no marketing fluff. Most posts needed minimal tone adjustment.

**Speed.** Eighteen posts, each reviewed, revised, and polished. The AI drafting was fast — a first draft in minutes. But the critique, revision, and manual review cycle added significant time per post. The total time was still a fraction of writing from scratch, but "AI writes it instantly" understates the real effort. The Copilot subscription cost is modest (included with GitHub Copilot Pro); the time cost of careful review is the real investment.

**Example provenance.** The examples in this series are sanitized composites — inspired by real scenarios from my production experience, genericized to prevent exploitation. No production data, client data, or proprietary scripts were pasted into AI prompts during the writing process.

## Fair Question: Did You Just Read AI Slop?

Parts of this series started as AI drafts. They stayed only if they survived manual review, technical correction, and my own judgment about what's accurate and useful for production DBAs.

Is that different from a human author who researches a topic, writes about it, and has an editor review it? Maybe not fundamentally. The difference is speed and scale — and the risk that the "research" step is replaced by pattern-matching on training data rather than production experience.

I think the disclosure matters. You should know how this content was produced so you can calibrate your trust accordingly. If a claim in this series doesn't match your experience, question it — just as you would with any technical blog post, AI-assisted or not.

## The Raw Materials

The entire working directory is available for inspection:

```text
C:\temp\dba-blog\
  plan.md              — Series outline and editorial guidelines
  process/
    log.md             — Chronological process log
    decisions.md       — Key editorial decisions and rationale
  drafts/
    post-01-blind-spot.md   through   post-18-meta.md
```

The process log captures planning, early post iterations, and editorial decisions. It's not a complete transcript of every conversation — those sessions don't persist between restarts — but it documents the key decisions, critiques, and revision patterns. The decisions file records why the series is named what it is, why the tone rules exist, and why each editorial choice was made.

## What This Demonstrates

If you've read seventeen posts and found them useful, informative, and technically credible — and you now learn they were coauthored with an AI — that's worth thinking about. Not as proof that AI content is trustworthy by default. Usefulness is not the same as accuracy, and plausible-sounding prose is exactly what untrustworthy AI content does well.

What it demonstrates is that AI agents, combined with domain expertise and rigorous review, can produce content that meets a professional standard. The AI didn't write this series alone. It couldn't have. It doesn't know what matters in production. It doesn't know which claims will make a senior DBA roll their eyes. It doesn't know my audience, my relationships in the SQL Server community, or the specific technical nuances that separate credible advice from plausible-sounding nonsense.

But it drafted fast. It structured well. It caught its own mistakes when asked to critique itself. And it freed me to focus on what I'm actually good at: knowing which advice is worth giving and which isn't.

That's the same value proposition this series has been describing for seventeen posts. The DBA's judgment stays essential. The tools just make it go further.

---

*This is the final post in the [ALTER DBA ADD AGENT](/ai-for-dbas/alter-dba-add-agent/) series. For the full post index, see the series landing page.*


---

*Part of the [ALTER DBA ADD AGENT](/ai-for-dbas/alter-dba-add-agent/) series — [Previous: The DBA Team](/ai-for-dbas/ai-augmented-dba-team/)*
