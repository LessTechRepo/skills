# Variables and Execute Parameters

Use this to decide what belongs in a Variable, how to read it, and how to override it for a single run.

## What Goes Where

**Use a Variable for:**

- API keys, passwords, and other secrets
- values that differ between assets or environments
- values a caller may want to override per run, such as a start date

**Keep in the script instead:**

- constants that never change
- logic
- anything that has to change during a run, because the SDK can only read variables

**To persist state between runs** (a cursor or last-run timestamp, for example), write it to a table. Nothing on disk survives a run, and scripts can't update variables.

## Defining Variables

- Define them in the UI: the Variables step when creating the asset, or later by editing the asset. There is no SDK call that sets a variable.
- Define them before running code that reads them. `get_variable` raises `less.LessException` if the name is missing.
- Values are read from the saved asset, so saving a change doesn't require a new token. They are cached per process, so restart the process or call `less.reload_config()` to see a change.

The form enforces these rules:

- Names must match `^[A-Za-z_][A-Za-z0-9_]*$` and be unique. Lookups are case-sensitive.
- Values are required, so an empty variable can't exist. To make something optional, leave it undefined and use a default.

## Reading Variables

```python
api_key = less.get_variable("API_KEY")                  # required: raises if missing
lookback = int(less.get_variables().get("LOOKBACK", 7))  # optional: has a default
```

- **Stored values are always strings.** Convert them with `int()`, `json.loads()`, or date parsing. `"false"` is truthy in Python, so compare explicitly.
- **Parameter values can be any JSON type.** A parameter can overwrite a variable with a number or boolean, so normalise before comparing: `str(value).lower() == "true"`.
- **Read config at the top of the script,** so a missing value fails before any work is done.
- **Variables are scoped to the asset.** A script can't read another asset's variables. Anyone with edit access to the asset can view the values.
- **Never log secret values.** Logging a non-sensitive setting, such as the start date in the template, is fine.

## Execute Parameters

Execute parameters override variables for a single run. They are sent in the API request that starts the run:

```bash
curl -X POST "$LESS_API_URL/public/v1/assets/$ASSET_ID/execute" \
  -H "Authorization: Bearer $API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"jobSource":"API","parameters":{"start_date":"2026-01-01"}}'
```

`$API_TOKEN` here is a Less API token, not the script token. Scripts can't start other runs.

| Function | Returns |
| --- | --- |
| `less.get_variable(name)`, `less.get_variables()` | The merged values, with parameters winning. Keys that exist only in the parameters are included. **Use these by default.** |
| `less.get_parameter(name)`, `less.get_parameters()` | Only what the request sent. `get_parameter` raises if the name wasn't sent. |

- **Local runs and scheduled runs have no parameters,** so `get_parameters()` returns `{}` locally. Code that requires a parameter-only name fails locally, so give it a default or also define it as a Variable.
- **A local credentials block left in the deployed script also empties parameters.** Its token isn't tied to a job. See [runtime.md](runtime.md#local-credentials).
- **Never send secrets as parameters.** The SDK writes the parameters to the job log in plain text the first time variables are read.
