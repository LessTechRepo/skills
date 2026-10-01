# Analysis Playbook

Use this to find tables and answer data questions.

## Steps

1. **Find the table.**
  - Know the name or topic: `searchAssets` (use `assetTypeFilter: table` to narrow). Multi-word queries fall back to per-token name matching, so try single distinctive words if a phrase finds nothing.
  - Know the producing source or model: `findTables` with its id. This is faster than `getLineage` for "which tables does X create?".
  - If `truncated` is true, say there are more (quote `total`) and offer a narrower query.
2. **Get the schema.** `getTableMetadata` returns exact column names, types, row counts, `tableStatus`, and `analyzable`. Use the exact column names it returns.
3. **Check** `analyzable`**.** `analyzeData` only works on verified tables. If `analyzable` is false, tell the user the table must be marked verified in Explore before it can be analyzed. You can still show a `getSampleData` peek.
4. **Peek at values (when it matters).** `getSampleData` returns the first 10 rows. Use it to learn categorical labels, date and number formats, and nulls before writing filters. It is a peek, not a query: do not infer totals or distributions from it.
5. **Run the query.** `analyzeData` with the tables and `SELECT`.



## `analyzeData` Tips

- Each table entry needs `tableId`, `assetId`, and an `alias`. For table assets `assetId` equals `tableId`. Reference tables in SQL by alias. Up to 10 tables per call, so joins are fine.
- Avoid `SELECT *`. Name columns.
- Prefer aggregates: `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`, `GROUP BY`. At most 25 rows return.
- If the result carries a `note` that it was capped, re-run with `COUNT`/`SUM`/`GROUP BY` or a tighter filter rather than reporting the capped rows as complete.
- For "top N", use `ORDER BY ... LIMIT N` with an explicit limit.
- Check filter values against `getSampleData` or a `SELECT col, COUNT(*) ... GROUP BY col` before relying on a string match.
- If a query errors on a column, re-check `getTableMetadata` rather than guessing names.
- Long text cells are shortened (`cellsTruncated`). Do not treat shortened values as full values.



## Reporting

- Lead with a concise answer to the question, then the supporting numbers.
- State the table(s) used and link them with the returned `tableLinks`.
- Mention caveats that affect the answer: truncation, unverified tables, nulls, or a date range you assumed.

