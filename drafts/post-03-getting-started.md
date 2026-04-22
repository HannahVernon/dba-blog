# Getting Started: Your First Hour with GitHub Copilot CLI

You've read the [why](/ai-for-dbas/the-dbas-blind-spot-why-ai-coding-agents-are-coming-for-your-workflow/) and the [what](/ai-for-dbas/what-can-an-ai-coding-agent-actually-do-for-a-dba/). Now let's get your hands on it.

This post walks you through installing GitHub Copilot CLI on Windows, running your first interaction, and getting a quick win — all in about an hour. By the end, you'll have used an AI agent to do something genuinely useful with a real stored procedure or script.

> **A note on installation steps:** CLI tools evolve fast. The steps below reflect the process as of mid-2026. If something doesn't match, check the [official Copilot CLI documentation](https://docs.github.com/en/copilot/github-copilot-in-the-cli) — the concepts in this post remain the same regardless of which install command you use.

## What You Need

- A **GitHub account** (free tier works)
- A **GitHub Copilot subscription** — Individual, Business, or Enterprise. If you're not sure whether your organization already has licenses, check with your IT team. There's also a free tier with limited usage if you want to try before committing.
- **PowerShell 7+** recommended (6+ minimum). [Windows Terminal](https://aka.ms/terminal) is a nice host for it but isn't required.

That's it. You don't need Visual Studio Code. You don't need to change your editor or workflow. The CLI runs in your existing terminal.

## Installation

The fastest path on Windows is WinGet:

```powershell
winget install GitHub.Copilot
```

Alternatively, if you prefer npm (requires Node.js):

```powershell
npm install -g @github/copilot
```

Then launch the CLI:

```powershell
copilot
```

On first launch, you'll be prompted to authenticate — this opens a browser window where you sign in with your GitHub account and authorize the app. You'll also be asked whether to trust your current working directory.

**Important:** start from a clean working directory that contains only the scripts you want the agent to see — not your profile folder, not a directory containing credentials or connection strings, and not a folder with production data exports. The agent reads files in the trusted directory, and anything it reads becomes part of the conversation context sent to GitHub's infrastructure. Treat this the same way you'd treat any cloud-connected tool: don't feed it secrets, customer data, or sensitive infrastructure details unless your organization has explicitly approved it.

Verify it's working by typing a simple prompt:

```text
What version of PowerShell am I running?
```

If you get back a sensible response, you're live.

## Your First Real Interaction

Don't start with "write me a backup script." Start with something where you can immediately verify the output.

First, navigate to a directory that contains a stored procedure or T-SQL script you're familiar with:

```powershell
cd C:\DBA\Scripts
copilot
```

The agent's context starts from this directory — it's where it looks when you reference files. If you don't have a script handy, create a file called `test-proc.sql` with any procedure you know well.

Then ask:

```text
Explain the stored procedure in test-proc.sql. What does it do, what tables does it read
from and write to, and are there any potential issues?
```

The agent reads the file, analyzes it, and gives you back a structured explanation. Compare it against what you know about the procedure. You'll likely find it catches things you'd forgotten — a table reference buried in dynamic SQL, a missing error handler, a `SELECT *` that pulls more columns than needed.

This is usually where the value becomes obvious. The agent didn't just autocomplete a line — it inspected your code and reasoned about it.

## How Context Works

The agent's context starts from the directory where you launched it. It can read files you reference — either by name in your prompt or by explicitly including them. When you say "look at test-proc.sql," it reads the file from disk.

**Practical tips:**

- Before starting a session, `cd` into the directory that contains the files you want to work with
- Reference specific files by name when your question is about particular code
- If you're working across multiple directories, you can point the agent at additional paths. For example: "Also look at the DDL files in C:\DBA\Schema for table definitions"
- The agent is most useful when it can see your actual code, schema files, or DDL extracts — not when it's guessing

## Custom Instructions: Teaching It Your Standards

Out of the box, the agent writes functional T-SQL — but it might not match your team's conventions. It'll use `CAST` instead of `CONVERT`, omit semicolons, or use bare `varchar` without a size.

You fix this once. Create a custom instructions file with your rules — something like:

```markdown
## T-SQL Standards
- Terminate every statement with a semicolon
- Always schema-qualify object references
- Always specify column lists in INSERT statements
- Always specify varchar/nvarchar size
- Use [square brackets] on all identifiers
- Use explicit ANSI JOINs, never implicit/comma joins
- Data types are always lowercase
```

Copilot considers these instructions automatically when generating code. This usually gets the output much closer to your standards, though you should still review what it produces — instructions are guidance, not enforcement. I've written about this in detail — including where the file lives and how to build it up over time — in [Teaching GitHub Copilot Your T-SQL Coding Standards](/tools/teaching-github-copilot-your-t-sql-coding-standards/).

## Common First-Session Mistakes

**Prompts too vague.** "Help me with SQL" gives you generic results. "Write a query against `sys.dm_exec_query_stats` that returns the top 10 queries by average elapsed time, with the query text and execution count" gives you something you can actually use.

**Not providing context.** If you ask about "the backup procedure" without being in a directory that contains one, the agent has nothing to work with. Either navigate to the right directory or reference the file explicitly.

**Starting in the wrong directory.** Launching from `C:\Users\you` means the agent can see your entire profile but probably not your SQL scripts. Launch from the directory where your work lives.

**Being surprised by permission prompts.** The agent will ask before running commands or accessing files outside its trusted scope. This is normal — it's a safety feature, not a bug.

**Expecting perfection on the first try.** The agent is iterative by design. Its first draft is a starting point — say "also add error handling" or "filter out system databases" and it refines. This back-and-forth is the normal workflow, not a failure.

**Not verifying the output.** The agent is confident and usually right, but it can hallucinate column names or misunderstand your schema. Always review what it produces against your actual environment before running it in production.

## Quick Win: Document an Undocumented Procedure

Here's something you can do right now that will save you real time. Pick a stored procedure on your system that has zero documentation — every DBA has at least a dozen of these.

Save the procedure definition to a `.sql` file, then ask:

```text
Read the procedure in legacy-proc.sql and generate documentation for it. Include:
a one-paragraph summary, parameter descriptions, tables referenced, return values,
and any side effects like INSERT/UPDATE/DELETE operations. Format as markdown.
```

You'll get back structured documentation you can drop into a wiki, a README, or a comment header. It won't be perfect — you'll want to verify the details and add business context the agent can't infer — but it gets you 80% of the way there in two minutes instead of an hour.

Save the output to a `.md` file and open it in your preferred markdown viewer (I use [GithubMarkdownViewer](https://github.com/HannahVernon/GithubMarkdownViewer)) for a clean formatted read.

---

That's your first hour. You've installed the CLI, authenticated, run a real analysis, set up your coding standards, and produced something useful. From here, the rest of the series digs into specific DBA tasks — each post assumes you have a working CLI session and know the basics.

**Try This Yourself:** Pick three undocumented procedures from your environment. Generate documentation for all three. See how long it takes compared to doing it manually — and notice what the agent catches that you might have missed.

**Next up:** [Writing T-SQL with an AI Partner](/ai-for-dbas/writing-t-sql-with-an-ai-partner/) — generating, reviewing, and refactoring T-SQL with AI assistance.


---

*Part of the [ALTER DBA ADD AGENT](/ai-for-dbas/alter-dba-add-agent/) series — [Previous: What Can an AI Agent Do?](/ai-for-dbas/what-can-an-ai-coding-agent-actually-do-for-a-dba/)*
