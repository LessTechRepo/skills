---
name: less-workspace
description: "Explore and operate a Less data workspace through the Less MCP server. Use to find sources, models, tables, orchestrations, and destinations; answer data questions with aggregate SQL over verified tables; understand how something is built (lineage, model pipelines, where a metric is computed) and what it affects; check pipeline health, diagnose failed jobs from logs, review schedules; and, only when explicitly asked, run or refresh an asset."
---

# Less Workspace

Use the Less MCP tools to explore a user's workspace, answer data questions, inspect pipelines, and check job health. Every call runs as the signed-in user and is limited to what they may read or execute.

Assume the `less` MCP connection is already configured by the client. Use the exact tool names the client exposes; prefixes may vary. Do not copy tool schemas from memory — rely on the tools' advertised descriptions.

## Core Rules

- **Never invent ids.** Resolve every asset or table id with `searchAssets` or `getAssetDetail` first (or `getTableMetadata` for tables).
- **Respect truncation and caps.** If a result has `truncated: true` or a `note`, the page is incomplete. Tell the user, quote `total`, and offer a narrower query. Never treat a page as the whole workspace.
- **Be economical.** Use the fewest calls that answer the question. Stop once the evidence is sufficient.
- **Permissions are real.** `Unauthorized`, "not found", or empty results can mean the user lacks access. Report that plainly; do not retry blindly or work around it.
- **Surface links.** Include the in-app links tool results return so the user can open the asset, run, or log. Never fabricate links.
- **Do not fabricate.** Report only what tool results show. Say so when data is missing.
- **Mutation safety.** `executeAsset` is the only tool that changes anything. See [Running assets](#running-assets).



## Choose a Workflow


| The user wants to…                                                   | Workflow               | Read                                                 |
| -------------------------------------------------------------------- | ---------------------- | ---------------------------------------------------- |
| Find assets, or get a number/trend/breakdown from data               | Answer a data question | [references/analysis.md](references/analysis.md)     |
| Know how something is built, where it comes from, or what it affects | Understand a pipeline  | [references/pipelines.md](references/pipelines.md)   |
| Know what failed, why something is stale, or refresh something       | Check health / operate | [references/operations.md](references/operations.md) |


Real questions often span workflows (for example, "why is this dashboard stale?" needs operations, then lineage, then analysis). Read the references you need and combine them; do not pick just one.

### 1. Answer a data question

`searchAssets` / `findTables` → `getTableMetadata` → (optional) `getSampleData` → `analyzeData` with an aggregate.

- Only run `analyzeData` on tables where `analyzable` is true.
- Prefer aggregate SQL (COUNT/SUM/GROUP BY). Returned rows are capped, so re-query with aggregates rather than assuming you saw everything.

### 2. Understand how something is built or what it affects

`getAssetDetail` → `getLineage` (upstream = provenance, downstream = impact) → `getModelPipeline` for a single model, or `searchPipelineSteps` to locate a computation across many models.

### 3. Check health, diagnose a failure, or refresh

`getJobRuns` (daily overview) → `getJobDetails` (specific runs) → `detailedLog` (log lines). If the user explicitly asks to refresh, run the pre-flight and then `executeAsset`.

## Running assets

Call `executeAsset` **only** when the user explicitly asks to run, refresh, or re-execute an asset. Do not run assets to "check", "test", or "help diagnose".

Before running:

1. Resolve the asset with `searchAssets` / `getAssetDetail`.
2. Confirm `canExecute` is true. If not, tell the user they cannot run it; do not try.
3. For models, confirm `hasPublishedVersion` is true. If not, the run will be rejected; tell the user it needs publishing first.
4. If several assets could match the request, confirm which one before running.

After running, follow the returned `jobId` (with the same `assetId`) via `detailedLog`. Do not look the run up with `getJobDetails` by date.

## Example Prompts

- "What tables in my workspace hold monthly order history?" (discovery)
- "Which five customers had the highest revenue last quarter?" (analysis)
- "What feeds the `revenue_daily` model, and what breaks if I change it?" (lineage)
- "Where in our models do we compute gross margin?" (pipeline step search)
- "What failed in the last 7 days, and why?" (operations)
- "Refresh the `orders` source." (explicit execute, with pre-flight)

## Out of Scope

- Creating, editing, or deleting assets: the MCP is read-mostly and only runs existing assets.
- Obtaining or storing tokens: authentication is handled by the client's MCP connection.
- Assumptions about a specific workspace's data: always discover it via the tools.

