# Pipelines and Lineage

Use this to explain how something is built, where data comes from, and what a change would affect.

## Pick the Right Tool

| Question | Tool |
| --- | --- |
| What is this asset? Description, last run, schedule | `getAssetDetail` |
| Which tables does this source/model write? | `findTables` |
| Where does this come from? What depends on it? | `getLineage` |
| What does this model do, step by step? | `getModelPipeline` |
| Where is X computed, across many models? | `searchPipelineSteps` |

Start from a resolved id (`searchAssets` / `getAssetDetail`). Never guess ids.

## Lineage

`getLineage` returns nodes and edges.

- `direction: upstream` is **provenance**: what feeds the asset. Use for "where does this come from?" and "why is this value wrong?".
- `direction: downstream` is **impact**: what consumes the asset. Use for "what breaks if I change/delete this?".
- Default is both. Narrow the direction when the question is one-sided to keep the graph small.
- `transactionDays` is the lookback window used to derive edges (default 7; 0 = all time). If an expected dependency is missing, widen the window before concluding it does not exist, for example for assets that run weekly or monthly.
- Describe the graph in order (source → model → table → destination) instead of dumping nodes and edges. Include links.

## Model Pipelines

- `getModelPipeline` lists a canvas model's steps in order (`toolId: summary`) plus a legend mapping `toolId` to name and description. `includeDraft` defaults to true; set it to false when the user cares about what is currently published.
- It returns at most the first 80 steps. For larger models, say the view is partial and use `searchPipelineSteps` to find the relevant part.
- Summarize the flow in plain language (inputs → joins/filters → calculations → outputs). Quote the specific step text for the part the user asked about.

## Finding Where Something Is Computed

`searchPipelineSteps` is semantic search over model steps ("where is revenue computed?").

- Scope it with either `modelIds` (up to 12) or `rootAssetId` plus `direction` to search every model reachable in that asset's lineage. Use one or the other.
- Hits include `modelId`, `modelName`, `nodeId`, and `stepText`. Name the model and the step, and link the model.
- Several hits may be legitimate (a metric defined in multiple models). Report all relevant ones and note any difference rather than picking one.

## Reporting

- Distinguish what you saw (tool results) from what you infer.
- For impact questions, list affected assets by type and note which are scheduled, since those will fail on the next run.
