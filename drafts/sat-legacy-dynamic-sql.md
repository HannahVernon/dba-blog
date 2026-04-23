Every SQL Server estate has at least one stored procedure that builds its query as a string. You open it, see `DECLARE @SQL nvarchar(MAX)` on line 3, and spend the next hour mentally tracing which `IF` blocks append which `WHERE` clauses. Dynamic SQL isn't inherently bad — `sp_executesql` with proper parameterization is a legitimate pattern — but the legacy version, built from raw string concatenation with no parameterization, is a different animal entirely.

AI agents are exceptionally good at untangling these procedures. They can trace variable assignments across hundreds of lines, reconstruct the final SQL for different parameter combinations, and flag injection risks that a tired DBA might miss on line 247 of a 300-line procedure. This is one of the places where AI saves the most time in [reverse-engineering legacy code](/ai-for-dbas/reverse-engineering-legacy-procedures/).

## The Problem: A Kitchen-Sink Search Procedure

Here's a pattern that exists in nearly every line-of-business application — a search procedure that builds its query dynamically based on which parameters the caller passes in. This is a simplified version of what I see in production:

```sql
CREATE OR ALTER PROCEDURE [dbo].[usp_SearchOrders]
    @CustomerName varchar(100) = NULL
    , @OrderDateFrom datetime = NULL
    , @OrderDateTo datetime = NULL
    , @StatusCode char(1) = NULL
    , @MinTotal decimal(18, 2) = NULL
    , @SortColumn varchar(50) = 'OrderDate'
    , @SortDirection varchar(4) = 'DESC'
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @SQL varchar(MAX);
    DECLARE @WhereClause varchar(MAX);

    SET @SQL = 'SELECT o.OrderID, o.OrderDate, o.OrderTotal, o.StatusCode'
        + ', c.CustomerName, c.AccountNumber'
        + ' FROM dbo.Orders o'
        + ' INNER JOIN dbo.Customers c ON o.CustomerID = c.CustomerID';

    SET @WhereClause = ' WHERE 1=1';

    IF @CustomerName IS NOT NULL
        SET @WhereClause = @WhereClause
            + ' AND c.CustomerName LIKE ''%' + @CustomerName + '%''';

    IF @OrderDateFrom IS NOT NULL
        SET @WhereClause = @WhereClause
            + ' AND o.OrderDate >= ''' + CONVERT(varchar(23), @OrderDateFrom, 121) + '''';

    IF @OrderDateTo IS NOT NULL
        SET @WhereClause = @WhereClause
            + ' AND o.OrderDate <= ''' + CONVERT(varchar(23), @OrderDateTo, 121) + '''';

    IF @StatusCode IS NOT NULL
        SET @WhereClause = @WhereClause
            + ' AND o.StatusCode = ''' + @StatusCode + '''';

    IF @MinTotal IS NOT NULL
        SET @WhereClause = @WhereClause
            + ' AND o.OrderTotal >= ' + CONVERT(varchar(20), @MinTotal);

    SET @SQL = @SQL + @WhereClause;

    /* Dynamic ORDER BY */
    IF @SortColumn = 'CustomerName'
        SET @SQL = @SQL + ' ORDER BY c.CustomerName';
    ELSE IF @SortColumn = 'OrderTotal'
        SET @SQL = @SQL + ' ORDER BY o.OrderTotal';
    ELSE
        SET @SQL = @SQL + ' ORDER BY o.OrderDate';

    IF @SortDirection = 'ASC'
        SET @SQL = @SQL + ' ASC';
    ELSE
        SET @SQL = @SQL + ' DESC';

    EXEC(@SQL);
END;
```

At first glance this looks straightforward. But there are multiple problems hiding in this code that are hard to spot when you're tracing the concatenation manually — especially in the production version, which usually has 30+ parameters and 400+ lines.

## The Prompt

```text
Analyze this stored procedure. It builds a dynamic SQL search query from
optional parameters. I need:

1. A plain-English explanation of what the final SQL would look like for
   a typical call with @CustomerName = 'Smith', @OrderDateFrom = '2024-01-01',
   and all other parameters NULL.
2. Every SQL injection vulnerability — show me how an attacker could exploit
   each one.
3. A rewritten version using sp_executesql with proper parameterization.
   Keep the dynamic ORDER BY (that can't be parameterized) but make it safe.
   Bracket all identifiers, use semicolons, and use CONVERT not CAST.
```

## What the Agent Produced

The agent immediately identified three injection vectors:

1. **`@CustomerName`** — directly concatenated into a `LIKE` clause. An input of `'; DROP TABLE dbo.Orders; --` would execute arbitrary SQL. This is the primary injection vector.
2. **`@StatusCode`** — directly embedded in quotes with string concatenation. It's `char(1)`, which limits the practical attack surface to a single character, but the *pattern* is still dangerous — if someone later widens this to `varchar(20)`, it becomes exploitable. Bad habits baked into code survive longer than anyone expects.
3. **`@SortColumn` and `@SortDirection`** — the existing procedure constrains these to known literals via `IF/ELSE` logic, so they're not injectable *as written*. But the agent correctly flagged the pattern as risky: the safety depends on the whitelist being maintained as new sort options are added. The rewrite should use `QUOTENAME` or an explicit whitelist to make the safety structural rather than incidental.

The agent rendered the final SQL for the sample call, making the concatenation result visible:

```sql
SELECT o.OrderID, o.OrderDate, o.OrderTotal, o.StatusCode
    , c.CustomerName, c.AccountNumber
FROM dbo.Orders o
INNER JOIN dbo.Customers c ON o.CustomerID = c.CustomerID
WHERE 1=1
    AND c.CustomerName LIKE '%Smith%'
    AND o.OrderDate >= '2024-01-01 00:00:00.000'
ORDER BY o.OrderDate DESC
```

Then it produced the parameterized rewrite:

```sql
CREATE OR ALTER PROCEDURE [dbo].[usp_SearchOrders]
    @CustomerName varchar(100) = NULL
    , @OrderDateFrom datetime = NULL
    , @OrderDateTo datetime = NULL
    , @StatusCode char(1) = NULL
    , @MinTotal decimal(18, 2) = NULL
    , @SortColumn varchar(50) = 'OrderDate'
    , @SortDirection varchar(4) = 'DESC'
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @SQL nvarchar(MAX);
    DECLARE @Params nvarchar(MAX);

    SET @SQL = N'SELECT o.[OrderID], o.[OrderDate], o.[OrderTotal], o.[StatusCode]
        , c.[CustomerName], c.[AccountNumber]
    FROM [dbo].[Orders] AS o
    INNER JOIN [dbo].[Customers] AS c
        ON o.[CustomerID] = c.[CustomerID]
    WHERE 1 = 1';

    /* Build parameterized WHERE clause */
    IF @CustomerName IS NOT NULL
        SET @SQL = @SQL + N'
        AND c.[CustomerName] LIKE N''%'' + @p_CustomerName + N''%''';

    IF @OrderDateFrom IS NOT NULL
        SET @SQL = @SQL + N'
        AND o.[OrderDate] >= @p_OrderDateFrom';

    IF @OrderDateTo IS NOT NULL
        SET @SQL = @SQL + N'
        AND o.[OrderDate] <= @p_OrderDateTo';

    IF @StatusCode IS NOT NULL
        SET @SQL = @SQL + N'
        AND o.[StatusCode] = @p_StatusCode';

    IF @MinTotal IS NOT NULL
        SET @SQL = @SQL + N'
        AND o.[OrderTotal] >= @p_MinTotal';

    /* ORDER BY cannot be parameterized — whitelist the allowed values */
    SET @SQL = @SQL + N'
    ORDER BY '
        + CASE @SortColumn
            WHEN 'CustomerName' THEN N'c.[CustomerName]'
            WHEN 'OrderTotal'   THEN N'o.[OrderTotal]'
            ELSE N'o.[OrderDate]'
          END
        + CASE WHEN @SortDirection = 'ASC' THEN N' ASC' ELSE N' DESC' END;

    SET @Params = N'@p_CustomerName varchar(100), @p_OrderDateFrom datetime'
        + N', @p_OrderDateTo datetime, @p_StatusCode char(1)'
        + N', @p_MinTotal decimal(18, 2)';

    EXEC [sys].[sp_executesql]
        @SQL
        , @Params
        , @p_CustomerName = @CustomerName
        , @p_OrderDateFrom = @OrderDateFrom
        , @p_OrderDateTo = @OrderDateTo
        , @p_StatusCode = @StatusCode
        , @p_MinTotal = @MinTotal;
END;
```

## What I Validated and Changed

The agent's rewrite was solid. A few things I checked:

1. **The `LIKE` pattern.** The agent parameterized the search value but kept the wildcard concatenation inside the dynamic SQL. This is correct — you can't parameterize the `%` wildcards themselves, but the user input is now safely passed through `sp_executesql`'s parameter binding. No injection risk.

2. **The `ORDER BY` whitelist.** The `CASE` expression only emits column references from a fixed set of known values. Any unexpected input falls through to the `ELSE` (OrderDate). This is the correct pattern — you cannot parameterize `ORDER BY` columns, so whitelisting is the safe alternative.

3. **`nvarchar` vs `varchar`.** The agent correctly switched to `nvarchar(MAX)` for the SQL string and parameter definition — `sp_executesql` requires Unicode strings. The original used `varchar`, which would cause a silent conversion or an error.

4. **Plan cache behavior.** The parameterized version generates reusable query plans. The original `EXEC()` version creates a new plan for every distinct SQL string — which means every unique parameter combination compiles a new plan, bloating the plan cache. This is a significant performance improvement beyond the security fix.

One thing I added that the agent didn't suggest: a `PRINT @SQL` debug mode controlled by a `@Debug bit = 0` parameter. When legacy dynamic SQL misbehaves, being able to see the rendered SQL is invaluable for troubleshooting.

## The Bigger Pattern

Dynamic SQL dissection is where AI agents earn their keep in legacy code work. The mechanical task — tracing string concatenation across dozens of `IF` blocks and variable assignments — is exactly what AI is good at. It doesn't lose its place on line 183. It doesn't accidentally skip a branch. It can reconstruct the final SQL for any combination of inputs instantly.

What the agent *can't* do is tell you whether the query is correct for your business. It can tell you what the SQL does; it can't tell you whether the results are what the users expect. That's where you combine the agent's technical analysis with conversations with the people who use the application. For the full approach to [writing and refactoring T-SQL with AI](/ai-for-dbas/writing-tsql-with-ai-partner/), including testing strategies, see the earlier post in this series.

## Try This Yourself

Find a dynamic SQL procedure in your environment — `sp_helptext` across your databases will turn up plenty. Feed it to the agent with this prompt:

```text
Analyze this stored procedure's dynamic SQL. Trace the string concatenation
and show me the final SQL for a typical set of parameter values. Identify
any SQL injection vulnerabilities. Then rewrite it using sp_executesql with
proper parameterization. Keep any ORDER BY or table-name logic that must
remain dynamic, but make it injection-safe using whitelisting.
```

Run both versions on a test system with identical parameters and compare the results. Pay special attention to edge cases — NULL parameters, empty strings, and values containing single quotes. The parameterized version should handle all of these safely; the original may not.

For the full legacy code reverse-engineering workflow, see [Reverse-Engineering Legacy Stored Procedures](/ai-for-dbas/reverse-engineering-legacy-procedures/).

---

*Part of the [ALTER DBA ADD AGENT](/ai-for-dbas/alter-dba-add-agent/) series.*
