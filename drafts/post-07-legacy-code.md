# Understanding Unfamiliar Code: Reverse-Engineering Legacy Procedures

You've just inherited a 2,000-line stored procedure called `usp_ProcessDailyBatch`. No documentation. No comments. The original developer left three years ago. Your manager needs to know what it does by Friday because they want to migrate it to a new system.

Every DBA has been here. For me, this is one of the most useful places to put an agent to work.

## "Explain This in Plain English"

Start simple. Drop the procedure into your working directory and ask:

```text
Read usp_ProcessDailyBatch.sql and explain what it does in plain English.
Structure the explanation as:
1. Purpose — what is this procedure's job?
2. Inputs — what parameters and tables does it read from?
3. Outputs — what tables does it write to?
4. Flow — step by step, what happens when it runs?
5. Concerns — anything that looks fragile, risky, or unusual?
```

The agent reads the entire procedure and produces a structured summary. For a 2,000-line proc, this might be a two-page explanation — but it's two pages you didn't have five minutes ago. And it's usually accurate enough to give you a solid starting point.

The key word is *starting point*. The agent can misinterpret complex business logic, especially when the meaning depends on data values it can't see. "This updates the status to 'C' when the total exceeds the threshold" is technically correct; understanding that 'C' means "completed" and the threshold is a contractual obligation requires domain knowledge the agent doesn't have.

## Dependency Mapping

Once you understand what the procedure does, you need to know what depends on it — and what it depends on.

```text
Analyze usp_ProcessDailyBatch.sql and list:
1. Every table it reads from (SELECT, JOIN, WHERE EXISTS, subqueries)
2. Every table it writes to (INSERT, UPDATE, DELETE, MERGE)
3. Every other stored procedure or function it calls
4. Any linked server references
5. Any dynamic SQL — show me what SQL it would likely render for typical
   parameter values (note: the agent infers this from static analysis of
   the string concatenation logic — it can't execute the code, so complex
   conditionals may produce inaccurate results)
Present this as a dependency map I could paste into documentation.
```

This is tedious work to do manually. The agent handles it quickly and can catch things you'd miss on a first read — the `EXEC` call buried on line 1,847, the linked server reference in an `OPENQUERY` wrapper, the dynamic SQL that builds a table name from a variable.

It will also miss things. Synonyms, cross-database references via three-part names, runtime object names assembled in dynamic SQL, temp tables populated by other procedures, `EXECUTE AS` context switches, encrypted modules it can't read, CLR procedure calls, and Service Broker activation procedures are all blind spots. Treat the output as a strong starting point, not a complete map.

Pointing the agent at your own T-SQL script libraries, such as [John Ness' SQL-Server-Scripts](https://github.com/JohnKNess/SQL-Server-Scripts), gives the agent real reference material to work with. Point it at your scripts folder and it can cross-reference dependencies across your entire codebase.

## Untangling Dynamic SQL

Dynamic SQL is where legacy code gets truly opaque. A stored procedure that builds SQL strings from concatenated variables, `QUOTENAME` calls, and conditional blocks is nearly impossible to read statically.

```text
This procedure builds dynamic SQL using string concatenation. Trace through
the logic and show me the actual SQL statements it would generate for these
scenarios:
1. @ReportType = 'Monthly', @IncludeArchive = 1
2. @ReportType = 'Daily', @IncludeArchive = 0
Show the SQL it appears to generate for each scenario, with the parameter values
substituted in.
```

The agent traces through the concatenation logic and produces the fully rendered SQL for each scenario. This is enormously helpful for understanding what the procedure *actually does* versus what it looks like it does.

A word of caution: for very complex dynamic SQL with deeply nested conditionals, the agent can get the logic wrong. Verify its output by adding `PRINT` statements to the procedure, running it on a non-production instance with known parameters, and comparing the actual generated SQL to what the agent predicted.

## Generating Test Cases

Here's the part that really changes the workflow. You have a legacy procedure you're afraid to touch because there are no tests.

```text
Based on your analysis of usp_ProcessDailyBatch.sql, generate test scenarios
that would verify its correctness. For each scenario, provide:
1. Description of what's being tested
2. Setup SQL — INSERT statements to create the test data
3. Execution — the EXEC call with specific parameters
4. Validation — SELECT statements that verify the expected outcome
Cover: happy path, empty input, duplicate records, boundary dates, and
the NULL handling in the @CustomerType parameter.
```

The agent generates a test suite based on its understanding of the procedure's logic. These won't be perfect — they can't be, because the agent doesn't fully understand the business rules — but they give you a first-pass safety net that didn't exist before.

With a test harness in place, you can refactor more safely. But go carefully — what looks like a simple cleanup can change behavior in subtle ways:

- Converting implicit joins to ANSI syntax is usually safe for inner joins, but can change semantics for outer joins when predicates move between `WHERE` and `ON`
- Eliminating a cursor changes the locking pattern from row-at-a-time to set-based, which may introduce blocking or deadlocks that the cursor never triggered
- Removing or restructuring `BEGIN TRAN` / `COMMIT` blocks changes transaction scope and rollback boundaries
- Any change to a trigger-firing table can cascade in ways the test harness doesn't cover

Run the tests after each change, and test under realistic concurrency — not just single-user SSMS execution.

**Try This Yourself:** Pick a stored procedure you inherited but never fully understood. Ask the agent to explain it and generate test cases. You'll learn things about the procedure that even the last code review missed.

## Trigger Archaeology

Triggers are the hidden landmines of SQL Server. A table might have `INSERT`, `UPDATE`, and `DELETE` triggers that fire additional triggers on other tables, creating cascading chains that are nearly impossible to trace by reading code alone.

```text
I have these trigger files in the triggers/ folder. Trace the full chain of
events that occurs when I INSERT a row into dbo.Orders. Show every trigger
that fires, what it does, and what additional triggers it might cause.
Present this as a numbered sequence.
```

The agent maps the cascade — `INSERT` on Orders fires `trg_Orders_Insert`, which updates `OrderAudit`, which fires `trg_OrderAudit_Update`, which calls `usp_NotifyFulfillment`. That kind of chain is what causes "we changed one table and everything broke" incidents.

Watch for SQL Server-specific trigger behavior the agent might not account for: `AFTER` vs `INSTEAD OF` semantics, the `nested triggers` server option, recursive trigger settings, and — critically — whether triggers handle multi-row operations correctly or assume single-row `INSERT`/`UPDATE`. Ask the agent to check for `inserted`/`deleted` table usage that only works for single rows.

## When the Agent Gets It Wrong

The agent excels at syntactic analysis — what tables are referenced, what the control flow looks like, what the dynamic SQL resolves to. It's weaker at semantic analysis — *why* the code does what it does, what the business meaning of status codes are, and whether the logic is actually correct for your use case.

One important habit: tell the agent to explicitly list its unknowns — unresolved object references, dynamic SQL it can't fully expand, status code meanings it's guessing at, and assumptions about parameter values. This turns silent hallucination into visible uncertainty.

You can also help the agent get things right by giving it as much ancillary context as possible. The more the agent can see, the fewer gaps it has to guess about:

- **Table definitions and related procedures** — if the proc calls other procs or references lookup tables, include those files in your working directory
- **Code from related repositories** — if the application tier calls this procedure, pointing the agent at the calling code helps it understand parameter expectations and return value handling
- **API documentation** — if the procedure feeds an API or is called by one, the API spec tells the agent what the contract looks like
- **Status code or lookup value tables** — a simple CSV or markdown file mapping codes to meanings eliminates an entire category of guesswork
- **Sample data or representative parameter values** — even a few rows of realistic data helps the agent reason about edge cases
- **Job step definitions** — if the procedure runs inside a SQL Agent job with multiple steps, the surrounding steps provide critical sequencing context
- **Agent-generated diagnostic queries** — I ask the agent to write queries it thinks would help it understand the code better. Things like "show me the distinct values in the Status column" or "what does the distribution of OrderType look like?" I tell it not to include personally identifying information in the diagnostic queries, then I run them manually against real data and paste the results back. This bridges the gap between static code analysis and runtime reality — the agent goes from guessing what `Status = 'P'` means to knowing there are four status values with specific distributions, which completely changes its analysis of the branching logic.

The agent can only work with what you give it. The difference between a vague analysis and an accurate one is often just a few extra reference files.

After the agent gives you its analysis, validate against the real system:

- Check `sys.sql_expression_dependencies` for references the agent may have missed — and let the agent write the query for you:

```text
Write a query against sys.sql_expression_dependencies that shows every object
referenced by usp_ProcessDailyBatch, including cross-database and linked server
references. Include the referencing and referenced object types.
```

- Run the procedure on a non-production instance with known inputs and compare results
- Remember that execution context matters — SQL Agent jobs, `EXECUTE AS`, session settings, and linked server configurations can all change behavior

If the procedure drives billing, financial posting, inventory, or compliance reporting, assume the first explanation is incomplete until you've validated it against real executions.  Again, you can ask the agent to write you queries it thinks would be helpful in validating its approach, then run them manually against test data, providing the results back to the agent for validation.

Use the agent to accelerate the mechanical parts of reverse-engineering. Pair it with conversations with the people who use the system — they know what status 'P' means even if the code doesn't say.

**A note on data governance:** legacy code analysis often involves pasting procedure code, schema definitions, and sometimes sample data into the agent. Be aware of your organization's policies on sharing code with cloud-hosted AI services. Genericize or redact sensitive business logic, PII column names, and environment-specific details before pasting. The agent doesn't need real customer names to analyze a `MERGE` statement.

## Deep Dives

Want to go deeper? These companion posts walk through specific scenarios in detail:

- [AI-Assisted Dynamic SQL Dissection](/ai-for-dbas/ai-dynamic-sql-dissection/) — Understanding complex dynamic SQL
- [AI-Assisted Dependency Mapping](/ai-for-dbas/ai-dependency-mapping/) — Mapping object dependencies
- [AI-Assisted Cursor Elimination](/ai-for-dbas/ai-cursor-elimination/) — Step-by-step cursor replacement

---

**Next up:** [Wait Stats, Deadlocks, and Blocking Chains: AI-Assisted Diagnosis](/ai-for-dbas/wait-stats-deadlocks-blocking/) — using AI to accelerate the troubleshooting workflow.


---

*Part of the [ALTER DBA ADD AGENT](/ai-for-dbas/alter-dba-add-agent/) series — [Previous: PowerShell Automation](/ai-for-dbas/powershell-automation/)*
