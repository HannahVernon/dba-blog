SQL injection is a solved problem in theory — use parameterized queries and you're safe. In practice, every legacy codebase I've worked with has at least a few stored procedures that build SQL strings by concatenating user input directly into `EXEC()` calls. They've been running in production for years. Nobody wants to touch them. And nobody has audited them since they were written.

I covered the injection surface scan at a high level in [Security Audits: Finding What You Missed](/ai-for-dbas/security-audits-finding-missed/), and we walked through a detailed rewrite in [the dynamic SQL satellite post](/ai-for-dbas/reverse-engineering-legacy-procedures/). This post focuses on the audit itself: systematically scanning your procedure library to find and prioritize the risky ones.

## The Problem: Finding the Needles

A mid-size database might have hundreds of stored procedures. Maybe 10% use dynamic SQL. Of those, maybe half are actually vulnerable — the rest use `sp_executesql` with proper parameterization. Manually reading through every one is a week-long project that nobody will ever schedule. An AI agent can triage the entire list in minutes.

But this is heuristic analysis, not a security scanner. The agent is doing pattern matching on source text — it can miss vulnerabilities hidden behind complex variable flow, and it can't see encrypted modules or CLR procedures at all. Treat the results as a triage list, not a definitive assessment. In particular, substring matching like `LIKE N'%sp_executesql%@%param%'` does not prove proper parameterization — it only suggests it. A procedure might contain `sp_executesql` with `@param` in a comment or in a non-parameterized context. Always verify flagged procedures manually before closing them out as safe.

## The Prompt

```text
I need to audit all stored procedures in this database for SQL injection
vulnerabilities. Query sys.sql_modules to get the procedure definitions,
then analyze each one. For each procedure, classify it as:

1. SAFE — no dynamic SQL, or uses sp_executesql with proper parameterization
2. REVIEW — uses dynamic SQL but the risk is unclear (e.g., concatenates
   variables but they may not be user-controlled)
3. HIGH RISK — concatenates parameter values directly into EXEC() or
   sp_executesql without parameterization

For HIGH RISK procedures, show:
- The procedure name
- Which parameters are concatenated into dynamic SQL
- Whether QUOTENAME is used (and whether it's used correctly — QUOTENAME
  only protects identifiers, not values)
- Whether any input validation exists (LEN checks, REPLACE of quotes, etc.)
- The specific lines/patterns that are vulnerable

Don't produce exploit examples. Focus on identifying the vulnerable patterns
and explaining the risk.
```

## What the Agent Produced

The agent generated a two-phase approach. First, a T-SQL scanning query to identify candidates:

```sql
/* Phase 1: Find procedures with dynamic SQL patterns */
SELECT
    OBJECT_SCHEMA_NAME(m.[object_id]) AS [SchemaName]
   ,OBJECT_NAME(m.[object_id])       AS [ProcedureName]
   ,CASE
        WHEN m.[definition] IS NULL
            THEN N'ENCRYPTED — cannot scan'
        WHEN m.[definition] LIKE N'%EXEC(%'
             AND m.[definition] LIKE N'%+%@%'
            THEN N'HIGH RISK — EXEC with concatenation'
        WHEN m.[definition] LIKE N'%sp_executesql%'
             AND m.[definition] NOT LIKE N'%sp_executesql%@%param%'
            THEN N'REVIEW — sp_executesql without parameterization'
        WHEN m.[definition] LIKE N'%sp_executesql%'
             AND m.[definition] LIKE N'%sp_executesql%@%param%'
            THEN N'LIKELY PARAMETERIZED — verify manually'
        WHEN m.[definition] LIKE N'%EXEC(%'
            THEN N'REVIEW — EXEC present, check manually'
        ELSE N'SAFE — no dynamic SQL detected'
    END AS [RiskLevel]
   ,CASE
        WHEN m.[definition] LIKE N'%QUOTENAME%'
            THEN N'Yes'
        ELSE N'No'
    END AS [UsesQuoteName]
   ,LEN(m.[definition])               AS [DefinitionLength]
FROM sys.[sql_modules] AS m
    INNER JOIN sys.[procedures] AS p
        ON m.[object_id] = p.[object_id]
WHERE m.[definition] LIKE N'%EXEC%'
    OR m.[definition] LIKE N'%sp_executesql%'
    OR m.[definition] IS NULL
ORDER BY
    CASE
        WHEN m.[definition] IS NULL THEN 1
        WHEN m.[definition] LIKE N'%EXEC(%'
             AND m.[definition] LIKE N'%+%@%' THEN 2
        ELSE 3
    END
   ,[SchemaName]
   ,[ProcedureName];
```

Then it recommended feeding each HIGH RISK procedure's full definition to the AI for detailed analysis. That's the right approach — the T-SQL scan identifies candidates, and the AI does the deep analysis.

## What I Changed

The scanning query was a reasonable first pass, but the `LIKE` patterns had issues:

- **False positives on `EXEC`.** `EXEC dbo.usp_SomeProcedure` is a normal procedure call, not dynamic SQL. The scan flagged any procedure that called another procedure. I narrowed it to `EXEC(@` and `EXEC (@` to catch the dynamic execution pattern specifically.
- **The `sp_executesql` parameterization check was too simple.** Checking for `@param` in the definition doesn't reliably distinguish parameterized from non-parameterized calls. I switched to checking whether the `sp_executesql` call has a second argument (the parameter definition string).
- **Missing the `EXECUTE` keyword.** Some developers write `EXECUTE(@SQL)` instead of `EXEC(@SQL)`. I added both variants.

For the deep analysis phase, I feed each flagged procedure to the agent with a targeted prompt:

```text
Analyze this stored procedure for SQL injection vulnerabilities. For each
parameter, trace whether its value flows into any dynamically executed SQL
through string concatenation. Identify:

1. Parameters concatenated directly into EXEC() or sp_executesql strings
2. Whether QUOTENAME is used — and note that QUOTENAME only protects
   identifier injection (table/column names), NOT value injection
3. Any REPLACE(@input, '''', '''''') patterns — these are insufficient
   because they don't protect against all injection vectors
4. Whether the procedure calls other procedures that might themselves
   be vulnerable (second-order injection)

Classify the overall risk and recommend the specific fix for each
vulnerability found. Do not produce exploit examples.
```

## The Key Patterns to Understand

When reviewing the agent's output, here's what matters:

**`EXEC(@SQL)` vs `sp_executesql`:** The `EXEC()` function provides no parameterization mechanism. If user input reaches the string, it's injectable. Always convert to `sp_executesql` with a parameter definition.

**`QUOTENAME` misuse:** I see this constantly — developers use `QUOTENAME` on a value parameter thinking it prevents injection. `QUOTENAME` wraps a string in square brackets (or quotes) to make it safe as an *identifier*. It does nothing for values in `WHERE` clauses. Safe identifiers need `QUOTENAME`; safe values need parameterized `sp_executesql`.

```sql
/* WRONG — QUOTENAME doesn't protect value injection */
SET @SQL = N'SELECT * FROM [dbo].[Orders]
    WHERE [CustomerName] = N''' + QUOTENAME(@CustomerName) + N'''';

/* RIGHT — parameterize the value, QUOTENAME the identifier */
SET @SQL = N'SELECT * FROM ' + QUOTENAME(@SchemaName) + N'.'
    + QUOTENAME(@TableName)
    + N' WHERE [CustomerName] = @p_Name';

EXEC sys.[sp_executesql]
    @SQL
   ,N'@p_Name nvarchar(100)'
   ,@p_Name = @CustomerName;
```

**The `REPLACE` false sense of security:** Some procedures do `SET @Input = REPLACE(@Input, '''', '''''')` before concatenating. This escapes single quotes, which blocks the simplest injection vector. But it doesn't protect against everything — Unicode normalization edge cases, second-order injection (where the escaped value gets stored and later concatenated without escaping), and non-string injection points (numeric values concatenated without quotes) all bypass this defense. Parameterization is the only reliable protection.

**Second-order injection:** Procedure A takes user input, escapes it, and stores it in a table. Procedure B reads that value from the table and concatenates it into dynamic SQL without parameterization. The agent can flag Procedure B's concatenation, but it can't automatically trace the data flow from A's stored value to B's dynamic SQL. This is where you need to manually verify the data lineage.

## The Final Workflow

Here's the complete audit process I use:

1. **Run the T-SQL scan** to get the candidate list.
2. **Feed each HIGH RISK procedure** to the agent for detailed analysis.
3. **Prioritize by exposure:** procedures called from web applications or APIs are higher risk than internal admin scripts.
4. **Remediate using `sp_executesql`** with proper parameterization. For the rewrite pattern, see the [dynamic SQL satellite post](/ai-for-dbas/reverse-engineering-legacy-procedures/).
5. **Test the rewrite** — run both versions with identical parameters, including edge cases: `NULL` values, empty strings, strings containing single quotes, and strings containing SQL keywords.

The agent won't find everything. Encrypted procedures, CLR modules, SQL Agent job steps, SSIS packages, and application code that builds SQL on the client side are all outside the scan's reach. But for the T-SQL surface area you can see, this approach covers it systematically.

## Try This Yourself

Start with the scanning query above. Run it against your busiest production database and see how many procedures light up as HIGH RISK. Then pick the worst offender — usually the oldest, most-modified search procedure — and feed its full definition to the agent.

The goal isn't to fix everything at once. It's to build a prioritized remediation backlog. The procedures that accept user-facing input and concatenate it into `EXEC()` are your critical findings. Internal procedures that concatenate system-generated values are lower risk (but still worth fixing — defense in depth).

For the broader security audit framework, see [Security Audits: Finding What You Missed](/ai-for-dbas/security-audits-finding-missed/). For the full reverse-engineering workflow that handles these rewrites, see [Reverse-Engineering Legacy Stored Procedures](/ai-for-dbas/reverse-engineering-legacy-procedures/).

---

*Part of the [ALTER DBA ADD AGENT](/ai-for-dbas/alter-dba-add-agent/) series.*
