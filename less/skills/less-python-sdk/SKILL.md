---
name: less-python-sdk
description: "Write, test, and deploy custom Python scripts on Less with the `less-sdk` package. Use when the user wants a Python asset or script that reads or writes Less tables, calls an external API from Less, uses `import less`, needs Variables or execute parameters, or hits a failing Python job. For exploring a workspace, answering data questions, or checking pipelines without writing a script, use `less-workspace`."
---

# Less Python SDK

Write custom Python scripts that run as Python assets on Less, using the `less-sdk` package. Scripts are developed and tested locally, then pasted into the asset in the Less UI. This skill describes `less-sdk` 0.3.0.

## Core Rules

1. **Only use the available libraries.** Check every import against the list below. A script that works locally can still fail on Less with `ModuleNotFoundError`.
   - **Available on Python 3.11:** the standard library, `pandas`, `numpy`, `pyarrow`, `requests`, `python-dateutil`, `pytz`, `tzdata`, and `less`.
   - **Not available:** `polars`, `duckdb`, `sqlalchemy`, `openpyxl`, `xlsxwriter`, `scipy`, `sklearn`, `matplotlib`, `boto3`, `psycopg2`, `httpx`, `bs4`, `lxml`, `yaml`.
   - Scripts can't declare dependencies. If a script truly needs something else, tell the user and don't work around it silently.
2. **Read secrets and config with `less.get_variable`.** Never hardcode them, and never use custom `os.environ` values, because they don't exist on Less.
3. **Log with `less.log`, not `print`.** `print` output doesn't reach the job log in Less.
4. **Remove the local credentials block before the script goes into Less.** If it stays, the job breaks about a day later. Prefer setting credentials in the environment, so there is no block to remove.
5. **Local runs are not a sandbox.** `less.write_table` writes real tables in the real workspace, even from a laptop. Confirm with the user before overwriting a table, or use a scratch table name.
6. **Pass only pandas DataFrames with supported dtypes to `write_table`.** Nullable and timezone-aware dtypes raise an error. See [references/tables.md](references/tables.md).

## Choose a Workflow

| The user wants to… | Workflow | Read |
| --- | --- | --- |
| Set up, write, test, or deploy a script | Follow the steps below | [references/runtime.md](references/runtime.md) |
| Handle API keys, config, or per-run overrides | Variables and execute parameters | [references/variables.md](references/variables.md) |
| Read or write Less tables | Tables, names, and dtypes | [references/tables.md](references/tables.md) |
| Fix a failing Python job | [Symptoms](#symptoms), then the job log | [references/runtime.md](references/runtime.md) |
| Explore a workspace or answer a data question, with no script | Use the `less-workspace` skill | |

## Write and Deploy a Script

1. **Install.** `pip install less-sdk`. Python 3.11 is required. To match Less exactly, use `pip install less-sdk==0.3.0 pandas==2.3.3 numpy==1.26.4 pyarrow==25.0.0 requests==2.34.2`.
2. **Create the asset.** The user creates a Python asset in the Less UI. The first step is **Variables**, so add variables there. Saving opens the **Local SDK setup** modal. The script editor only appears once the asset has been saved.
3. **Get credentials.** The user copies the generated script from the modal. To get it again: ⋯ menu → **Generate local SDK setup**. The token lasts 24 hours and then has to be regenerated.
4. **Develop and test locally.** Set `LESS_API_URL`, `LESS_API_TOKEN`, and `LESS_ASSET_ID` in the shell. See [Local credentials](references/runtime.md#local-credentials).
5. **Deploy.** Paste the script into the asset's Script step in the UI, without the local credentials block.
6. **Run.** Add a schedule, or start a run through the API. See [Execute parameters](references/variables.md#execute-parameters).

## Template

```python
from datetime import date, timedelta

import pandas as pd
import requests

import less

# Config first, so a missing value fails before any work is done
API_KEY = less.get_variable("API_KEY")
config = less.get_variables()
lookback_days = int(config.get("LOOKBACK_DAYS", 7))
start_date = str(config.get("START_DATE", date.today() - timedelta(days=lookback_days)))

less.log(f"Fetching rows from {start_date}")

response = requests.get(
    "https://api.example.com/orders",
    headers={"Authorization": f"Bearer {API_KEY}"},
    params={"since": start_date},
    timeout=60,
)
response.raise_for_status()

df = pd.DataFrame(response.json())
if df.empty:
    less.warning("No new rows, nothing written")
else:
    df["created_at"] = pd.to_datetime(df["created_at"], utc=True)
    table_id = less.write_table("orders", df)
    less.log(f"Wrote {len(df)} rows to {table_id}")
```

## Symptoms

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `ModuleNotFoundError` on Less | The library isn't available | Rewrite with the available libraries (rule 1) |
| Fails with 401 about a day after deploying | The local credentials block was left in | Remove the block and redeploy |
| Parameter comes back empty | Local and scheduled runs have no parameters, or the local block was left in | Use `less.get_variables()` with a default ([variables](references/variables.md#execute-parameters)) |
| `less.LessException` from `get_variable` | The name isn't defined, or differs in case | Add the Variable to the asset in the UI |
| `TypeError` from `write_table` | Unsupported dtype, or not a DataFrame | Convert the column ([dtypes](references/tables.md#dtypes-on-write)) |
| Odd column name, such as `Bel_b__kr_` | Names are rewritten on write | Rename before writing ([names](references/tables.md#names-are-rewritten-on-write)) |
| `PermissionError` writing a file | The working directory isn't writable | Use `tempfile` ([disk](references/runtime.md#disk)) |
| Job succeeds despite an error | `except Exception:` swallowed the stop, or `continue_on=True` | Re-raise `less.LessException` ([failure](references/runtime.md#logging-and-failure)) |

## Working with the Less MCP Tools

An agent with the Less MCP tools can support this work. See the `less-workspace` skill for how to use them.

- **Before writing a script:** use `searchAssets`, `getTableMetadata`, and `getSampleData` to find table ids and check schemas.
- **When debugging a run:** use `getJobRuns`, `getJobDetails`, and `detailedLog` to read what happened.
- **To run an asset:** call `executeAsset` only when the user explicitly asks.

## Example Prompts

- "Write a Less Python script that pulls orders from our billing API into a table." (new script)
- "Where do I put an API key in my Less script?" (Variables)
- "Can I use polars in a Python asset?" (libraries)
- "Why is my column called `Bel_b__kr_`?" (table names)
- "My Python job fails with `ModuleNotFoundError`." (debugging)

## Out of Scope

- Creating or editing assets for the user: the asset, its Variables, its script, and its schedule are all managed in the Less UI.
- Exploring a workspace or answering data questions without a script: use `less-workspace`.
