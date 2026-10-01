# Operations Runbook

Use this to check pipeline health, diagnose failures, review schedules, and (only on explicit request) run assets.

## Drill-Down

Go from broad to specific. Stop as soon as you can answer.

1. **Overview:** `getJobRuns`**.** Daily counts grouped by status over `from`/`to`. Filter with `status` (`success|warning|failed`) and `type` (`source|model|orchestration`). This is aggregate only; use it to find which days look bad.
2. **Runs:** `getJobDetails`**.** Individual runs in a date range with `jobId`, `assetId`, asset name, status, timing, and `logLink`. Pass `assetId` whenever you know it. Results are capped, so narrow the range or status instead of paging blindly.
3. **Logs:** `detailedLog`**.** Needs `assetId` + `jobId`. Returns up to 40 lines plus a link to the full log.
  - Default level is `error`, which is right for "why did it fail?".
  - Use `level: all` for timing, per-step, or "what did it do?" questions.
  - If the error log is empty for a failed run, retry with `warning`, then `all`.

Choose date ranges from the user's wording ("last week", "yesterday"), and state the exact dates you used.

## Schedules

`getSchedules` lists every scheduled asset (id, name, cron, timezone, next run). It takes no input and is capped, so say if it was truncated.

- Use it for "what runs when?", "when does X next run?", and to spot overlapping or missing schedules.
- Present cron in plain language with the timezone.



## Diagnosing a Failure

1. Find the failing run (`getJobRuns` → `getJobDetails`, filtered by `status: failed`).
2. Read its log (`detailedLog`).
3. If the cause is upstream, use `getLineage` (upstream) from the failing asset and check whether a dependency failed or was late. See [pipelines.md](pipelines.md).
4. Report: what failed, when, the error in the user's terms, the likely cause, and links to the run and log. Say clearly when the cause is a guess.

Do not re-run the asset to investigate. Only offer to run it, and wait for the user to ask.



## Stale Data

"Why is this table/dashboard out of date?"

1. Resolve the table, then find the asset that writes it (`getLineage` upstream, or `getAssetDetail` on the producer).
2. Check the producer's `lastJob` and schedule (`getAssetDetail`, `getSchedules`).
3. If the last run failed, diagnose it as above. If it succeeded but is old, check whether it is scheduled at all.



## Executing an Asset

`executeAsset` starts a real job. It is the only mutation in the toolset.

**Only when the user explicitly asks to run, refresh, or re-execute an asset:**

1. Resolve the asset via `searchAssets` / `getAssetDetail`. Confirm which asset if it is ambiguous.
2. Check `canExecute` from `getAssetDetail`. If false, stop and explain.
3. For a model, check `hasPublishedVersion`. If false, stop and explain that it needs to be published.
4. Call `executeAsset`. Report `success` and `message` honestly.
5. On success, follow the returned `jobId` with the same `assetId` via `detailedLog` (use `level: all` to show progress). Do not use `getJobDetails` by date to find this run.
6. Report the final state and link the run. If it fails, diagnose it as above, and do not automatically run it again.

Do not run downstream assets on the user's behalf unless they ask. Mention that downstream assets may need a refresh.