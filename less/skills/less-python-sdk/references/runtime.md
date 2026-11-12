# Runtime

Use this for local setup, available libraries, logging, and the limits a script runs under.

## Local Credentials

**Prefer environment variables.** Set these three values in the shell, and leave them out of the script. The SDK reads them when `less` is imported, so the same file then runs unchanged in both places, with nothing to strip and no token to leak.

```bash
export LESS_API_URL="https://app.less.tech/api"
export LESS_API_TOKEN="<script-api-token>"
export LESS_ASSET_ID="<python-asset-id>"
```

To keep them in an untracked `.env` file, load it into the shell with `set -a; source .env; set +a`. Don't load it from the script.

The Local SDK setup modal generates this block instead:

```python
# LOCAL ONLY — the os.environ block below is for running on your machine.
import os

os.environ["LESS_API_URL"] = "https://app.less.tech/api"
os.environ["LESS_API_TOKEN"] = "<script-api-token>"
os.environ["LESS_ASSET_ID"] = "<python-asset-id>"

import less

less.reload_config()  # pick up env if less was already imported

# END LOCAL ONLY
```

- **If the block is used, the script still needs its own imports.** The block contains `import less`, so the code that remains after removing it must import `less` and `pandas` itself. Check this before pasting.
- **`less.reload_config()` is only needed if `less` was imported before the variables were set.** This is common in notebooks.
- **Why leaving the block in is harmful:** it replaces the job's own credentials with the local token. That token isn't tied to a job, so execute parameters silently come back empty, and the script starts failing with 401 errors once the token expires, usually on a scheduled run a day later.
- **Treat the token as a secret.** Don't commit it, print it, or ask the user to paste it into chat if they can set it in the environment.
- **Add `.less_logs/` to `.gitignore`.** The SDK writes logs there during local runs. Ignore `.env` too, if one is used.

## Available Libraries

Less runs scripts on **Python 3.11** with the standard library plus the libraries below. These versions describe SDK 0.3.0 and may go stale.

| Library | Version |
| --- | --- |
| `pandas` | 2.3.3 |
| `numpy` | 1.26.4 |
| `pyarrow` | 25.0.0 |
| `requests` | 2.34.2 |
| `python-dateutil` | 2.9.0 |
| `pytz`, `tzdata` | 2026.3 |
| `less` (the SDK) | 0.3.0 |

**Not available:** `polars`, `duckdb`, `sqlalchemy`, `openpyxl`, `xlsxwriter`, `scipy`, `sklearn`, `matplotlib`, `boto3`, `psycopg2`, `httpx`, `bs4`, `lxml`, `yaml`.

- There is no requirements field on a Python asset, so scripts can't declare dependencies.
- Use the stdlib or the listed libraries instead: `requests` rather than `httpx`, `json` or `csv` rather than `yaml`, and pandas rather than polars.
- If a script truly needs something else, tell the user and don't work around it silently.
- Don't list or rely on anything else you find installed in the runtime. Other packages there aren't a supported part of it.

## Logging and Failure

```python
less.log("message")                          # info
less.info("message")
less.debug("message")
less.warning("message")                      # continues
less.warning("message", continue_on=False)   # stops the script
less.error("message")                        # stops the script
less.error("message", continue_on=True)      # logs and continues
```

- **Messages must be strings.** `less.log(df)` and `less.log(123)` raise `TypeError`, so use an f-string.
- **Stopping raises `less.LessException`, a subclass of `RuntimeError`.** A broad `except Exception:` swallows it and the job can then end as successful. Catch specific exceptions, or re-raise:

  ```python
  try:
      run()
  except less.LessException:
      raise
  except Exception as exc:
      less.error(f"Unexpected failure: {exc}")
  ```

- **Job status comes from the exit code.** Exit code 0 is success, even if `less.error(..., continue_on=True)` was logged. An uncaught exception or a non-zero `sys.exit` is a failure, and the traceback appears in the job log.
- **Don't call `less.send_logs`, `less.send_execution_info`, `less.configure`, or `less.logger` from scripts.** The platform handles those.

## Limits

### Disk

The working directory isn't writable, so a relative `open("out.csv", "w")` raises `PermissionError`. Use the `tempfile` module. Nothing on disk survives the run.

```python
import tempfile
from pathlib import Path

import pandas as pd

import less

with tempfile.TemporaryDirectory() as tmp:
    path = Path(tmp) / "export.csv"
    path.write_text("id,amount\n1,9.50\n")
    df = pd.read_csv(path)

less.log(f"Parsed {len(df)} rows")
```

### Everything Else

- **One file:** the asset stores a single script, so there are no local imports and no companion files.
- **Environment:** only the platform's own variables are present. Read everything else with `less.get_variable`.
- **No automatic retries:** neither the SDK's HTTP calls nor a failed run are retried. Add retries around flaky external APIs yourself, and make scripts safe to re-run. Use replace rather than `append` where possible, because re-running an append duplicates rows.
- **Time:** a run's credentials last 6 hours, so SDK calls after that fail, including a final `write_table`. Runs are stopped outright at 24 hours. Keep runs well under 6 hours.
- **Memory:** `read_table` and `append=True` load whole tables into memory.
- **API scope:** the SDK can only reach table, log, and variable operations for its own asset. A script can't manage other assets or start other runs.
