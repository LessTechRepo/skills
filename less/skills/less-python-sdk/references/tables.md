# Tables

Use this when a script reads or writes Less tables.

| Function | Behaviour |
| --- | --- |
| `less.list_tables()` | DataFrame with columns `id` and `table_name`, for tables the user can read, sorted by name. |
| `less.read_table(table_id)` | Downloads the whole table into a DataFrame. There is no column or row filtering. |
| `less.write_table(name, df, *, table_id=None, append=False)` | Writes the table and returns its id. `table_id` and `append` are keyword-only. |

## Finding a Table

Table names aren't unique, so find tables by id. Storing the id in a Variable works well. If filtering `list_tables()` by name, handle zero or several matches:

```python
tables = less.list_tables()
matches = tables[tables["table_name"] == "orders"]
if len(matches) != 1:
    less.error(f"Expected one table named orders, found {len(matches)}")

orders = less.read_table(matches.iloc[0]["id"])
```

Docs examples use `tables.iloc[0]` as a demo only. It picks whichever table sorts first, and it raises `IndexError` if the list is empty.

## Writing a Table

Local runs write to the real workspace too, so confirm with the user before overwriting a table, or use a scratch table name.

- **Default write:** replaces the table this asset owns under `name`, or creates it on the first run. Tables from other assets with the same name are not affected.
- **`table_id=`:** overwrites that specific table, which can belong to another asset, if the user has write access. The original name is kept, and a mismatched `name` logs a warning.
- **`append=True`:**
  - Stacks the new rows onto the existing rows, with no deduplication, so re-running an append duplicates rows.
  - Columns present on only one side are filled with nulls, and a warning is logged.
  - It reads and rewrites the whole table each time, so cost grows with table size.
  - With no `table_id` and no existing table, it creates the table.
- **Empty DataFrame:** nothing is written and only a warning is logged. The call returns `None`, or `table_id` if one was given. Check for this if later steps need the id.
- **The DataFrame index is not saved.** Call `reset_index()` first if it holds data.
- **Anything that isn't a pandas DataFrame** raises `TypeError`.

## Names Are Rewritten on Write

Non-ASCII and non-alphanumeric characters are replaced with `_`, so rename columns deliberately before writing.

| Input | Result |
| --- | --- |
| Table `My Orders / 2026!` | `My_Orders_2026_` |
| Column `Order Id` | `Order_Id` |
| Column `Order-Id`, alongside `Order Id` | `Order_Id_1` |
| Column `Beløb (kr)` | `Bel_b__kr_` |
| Column named `None` | `col_N`, where N is its position |

```python
df = df.rename(columns={"Beløb (kr)": "amount_dkk"})
```

## Dtypes on Write

| Input dtype | Result |
| --- | --- |
| `int*`, `uint*` | Integer |
| `float32`, `float64` | Decimal |
| Integers with nulls | Decimal (pandas makes them `float64`, so `1` becomes `1.0`) |
| `bool` | Boolean |
| `string`, `object`, `category` | Text |
| `datetime64[*]` | Datetime |
| UTC-aware datetime | Datetime, converted to naive UTC |
| `timedelta64` | Text, `HH:MM:SS` |
| Mixed `object`, `date`, `Decimal`, dict | Text, stringified: `1` → `'1'`, a dict → `"{'x': 1}"`, `date` → `'2026-01-01'`, `Decimal` → `'1.5'` |
| `object` booleans with nulls | Text: `'True'` |
| **`Int64`, `Float64`, `boolean`, `float16`** | **`TypeError`** |
| **Datetime with a non-UTC timezone** | **`TypeError`** |

Fixes for the errors and silent conversions:

```python
df["ts"] = df["ts"].dt.tz_convert("UTC")                       # non-UTC timezone
df["n"] = df["n"].astype("float64")                            # nullable Int64 or Float64
df["day"] = pd.to_datetime(df["day"])                          # Python dates, to keep a Datetime column
df["amount"] = df["amount"].astype("float64")                  # Decimal objects, to keep a number column
```
