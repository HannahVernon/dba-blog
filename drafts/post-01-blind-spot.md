# The DBA's Blind Spot: Why AI Coding Agents Are Coming for Your Workflow

You've probably used ChatGPT to explain an error message. Maybe you've tried GitHub Copilot's autocomplete in Visual Studio, or you've been using Redgate SQL Prompt for years. Those are useful tools. But they're not what I'm writing about.

What I'm writing about is something fundamentally different — and most DBAs I talk to haven't seen it yet.

## Code Completion vs. Coding Agents

**Code completion** predicts what you're about to type. SQL Prompt fills in column names. IntelliSense suggests table aliases. ChatGPT gives you a code block you copy into SSMS. In all of these, *you* drive — the tool reacts to your keystrokes or answers a question in isolation.

**A coding agent** operates in a loop: it inspects your environment, takes an action, checks the result, and revises. It reads your files. It runs commands in your terminal. It sees the output, decides if something went wrong, and tries a different approach.

When I point GitHub Copilot CLI at a stored procedure and say "optimize this," it reads the procedure, proposes changes, and iterates. In a controlled workflow where I provided execution plans and runtime statistics, it [cut logical reads by 73% and memory grants by 80%](/t-sql/can-github-copilot-optimize-your-t-sql-i-put-it-to-the-test/) on one production procedure — not by being smarter than me, but by being tireless in a way I can't be at 2pm on a Tuesday. One case study doesn't prove it'll do that every time, but it shows the potential.

Completion helps you type faster. An agent helps you *work* faster.

## What This Means in Practice

Most DBAs I know are stretched thin. You're managing backups, reviewing change scripts, troubleshooting blocking chains, documenting procedures that should have been documented three years ago, and answering the same questions from developers every week.

Here's what I've used a coding agent for recently:

- Reverse-engineered a 1,500-line inherited stored procedure — plain-English summary, data flow, every table it touches — in about two minutes
- Generated a PowerShell script to audit TLS certificates across 40 instances, with error handling and CSV export
- Fed actual deadlock XML from production into the agent and got back a hypothesis about the root cause with a suggested investigation path

None of these required genius. They required time, attention to detail, and familiarity with system views I've accumulated over decades. The agent can surface patterns and synthesize information quickly — and doesn't get interrupted by a Teams message halfway through. It's also confidently wrong sometimes, which is why every output gets reviewed before I act on it.

## The Leverage Shift

A less experienced DBA who learns to use these tools well can deliver routine scripting and documentation faster than an experienced DBA doing everything by hand. Not because the junior understands SQL Server better — they don't, and they won't for years. But the agent accelerates the mechanical parts of the job.

The senior's deep understanding still matters — arguably more than ever. It matters for architecture decisions, for capacity planning, for knowing when the agent's suggestion is subtly wrong, and for all the judgment calls that come with production operations. The agent is a force multiplier for execution, but it doesn't replace the experience that keeps you from deploying something that looks correct but fails under concurrency or parameter variation.

In my experience, what's already happening is this: DBAs who use these tools are getting more done with less effort.

## What's Coming in This Series

This is the first post in **ALTER DBA ADD AGENT**. Over the coming posts, I'll show what AI coding agents look like in real DBA work: T-SQL writing and review, automation, troubleshooting, security, monitoring, and tools like [Erik Darling's PerformanceMonitor](https://github.com/erikdarlingdata/PerformanceMonitor) that have AI integration built in from the ground up. Real prompts, real output, real tradeoffs.

**Next up:** [What Can an AI Coding Agent Actually Do for a DBA?](/ai-for-dbas/what-can-an-ai-coding-agent-actually-do-for-a-dba/) — a tour of every task area where these tools add value.


---

*Part of the [ALTER DBA ADD AGENT](/ai-for-dbas/alter-dba-add-agent/) series.*
