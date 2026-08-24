# pytest for ETL — A Practice Guide from Zero to Production

Work through this top to bottom. Each level introduces **one new pytest feature**, gives you an **exercise**, then the **solution**. Don't read the solution first — write your version, run it, watch it fail, then compare.

Every function under test is one I've seen break a real pipeline.

---

## Table of contents

| Level | Topic | New pytest feature |
|---|---|---|
| 0 | Setup & project layout | discovery rules |
| 1 | Your first test | `assert`, naming |
| 2 | Many inputs, one test | `@pytest.mark.parametrize` |
| 3 | Testing failure paths | `pytest.raises` |
| 4 | Money and floats | `pytest.approx`, `Decimal` |
| 5 | Reusable test data | `@pytest.fixture` |
| 6 | Factory fixtures | fixtures that return functions |
| 7 | File ingestion | `tmp_path` |
| 8 | Config & environment | `monkeypatch` |
| 9 | Mocking an API source | `unittest.mock`, `caplog` |
| 10 | Databases & idempotency | scoped fixtures, teardown |
| 11 | Spark setup & schema contracts | `conftest.py`, session scope |
| 12 | Joins, nulls, fan-out | `chispa` |
| 13 | Dedup & late-arriving data | window functions under test |
| 14 | SCD Type 2 | multi-step state tests |
| 15 | Timezones & partitions | frozen time |
| 16 | Data quality rule suite | parametrizing over rules |
| 17 | Capstone: end-to-end pipeline | markers, integration tests |

---

## Level 0 — Setup

```bash
mkdir etl-practice && cd etl-practice
python -m venv .venv && source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install pytest pandas pyspark chispa duckdb
```

### Project layout

```
etl-practice/
├── pyproject.toml
├── etl/
│   ├── __init__.py
│   ├── parsing.py
│   ├── transforms.py
│   └── spark_jobs.py
└── tests/
    ├── conftest.py
    ├── test_parsing.py
    └── test_transforms.py
```

Create the `__init__.py` — empty file, but without it your imports will fail confusingly.

### pyproject.toml

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = ["test_*.py"]
python_functions = ["test_*"]
addopts = "-v --strict-markers"
markers = [
    "slow: tests that take more than a second",
    "spark: requires a SparkSession",
    "integration: touches a real external system",
]
filterwarnings = ["ignore::DeprecationWarning"]
```

`--strict-markers` means a typo'd marker becomes an error instead of silently doing nothing. Turn it on now and save yourself an afternoon later.

### Discovery rules — the three things that trip up beginners

1. Files must be named `test_*.py` (or `*_test.py`).
2. Functions must be named `test_*`.
3. Classes must be named `Test*` **and have no `__init__` method**.

If pytest says "collected 0 items", you broke one of those three.

### The commands you'll actually use

```bash
pytest                          # everything
pytest -v                       # one line per test
pytest tests/test_parsing.py    # one file
pytest -k "email"               # only tests with "email" in the name
pytest -x                       # stop at first failure
pytest --lf                     # rerun only last failures
pytest -m "not slow"            # skip tests marked slow
pytest -q                       # quiet
pytest -s                       # let print() through
```

`pytest --lf -x` is the loop you'll live in while fixing a broken pipeline.

---

## Level 1 — Your first test

### Concept

A test is a function that runs some code and `assert`s something about the result. No framework classes, no `self`, no `assertEqual`. Just `assert`.

pytest rewrites the assert statement so that when it fails you see both sides:

```
E       assert 'A@X.COM' == 'a@x.com'
E         - a@x.com
E         + A@X.COM
```

### Code under test

```python
# etl/parsing.py
def normalize_email(value):
    """Trim, lowercase, and convert blanks to None."""
    if value is None:
        return None
    cleaned = value.strip().lower()
    return cleaned or None
```

### Exercise 1

Write four tests for `normalize_email`:

1. Leading/trailing whitespace is removed.
2. Uppercase is lowered.
3. `None` in → `None` out.
4. A string of only spaces (`"   "`) → `None`, **not** `""`.

Case 4 is the one that matters. Empty strings sneak into warehouses and then `WHERE email IS NULL` misses them forever.

<details>
<summary><b>Solution 1</b></summary>

```python
# tests/test_parsing.py
from etl.parsing import normalize_email


def test_strips_whitespace():
    assert normalize_email("  bob@x.com  ") == "bob@x.com"


def test_lowercases():
    assert normalize_email("BOB@X.COM") == "bob@x.com"


def test_none_passes_through():
    assert normalize_email(None) is None


def test_blank_string_becomes_none():
    assert normalize_email("   ") is None
```

Run it:

```bash
pytest tests/test_parsing.py -v
```

Now deliberately break the function — delete the `or None` — and rerun. Read the failure output carefully. Learning to read pytest failures is 80% of the value.

**Note:** use `is None`, not `== None`. `assert x is None` will catch a function that returns `float('nan')` or an empty numpy array; `==` may not.
</details>

---

## Level 2 — Many inputs, one test

### Concept

`@pytest.mark.parametrize` runs the same test body once per row of a table. Each row is reported as a **separate test**, so one bad input doesn't hide the others.

Without it you write six near-identical functions. With it you write one and a table.

### Code under test

```python
# etl/parsing.py
def parse_amount(value):
    """Parse a money string into a float.

    Handles: currency symbols, thousands separators, accounting
    negatives '(500)', blanks -> None.
    """
    if value is None:
        return None
    s = str(value).strip()
    if not s:
        return None

    negative = s.startswith("(") and s.endswith(")")
    if negative:
        s = s[1:-1]

    s = s.replace("$", "").replace(",", "").replace(" ", "")
    if not s:
        return None

    result = float(s)
    return -result if negative else result
```

### Exercise 2

Parametrize a single test covering:

| input | expected |
|---|---|
| `"1234.56"` | `1234.56` |
| `"1,234.56"` | `1234.56` |
| `"$1,234.56"` | `1234.56` |
| `"(500)"` | `-500.0` |
| `"($1,500.00)"` | `-1500.0` |
| `""` | `None` |
| `"   "` | `None` |
| `None` | `None` |
| `"0"` | `0.0` |

Give each case a readable id.

<details>
<summary><b>Solution 2</b></summary>

```python
import pytest
from etl.parsing import parse_amount


@pytest.mark.parametrize(
    "raw,expected",
    [
        ("1234.56", 1234.56),
        ("1,234.56", 1234.56),
        ("$1,234.56", 1234.56),
        ("(500)", -500.0),
        ("($1,500.00)", -1500.0),
        ("", None),
        ("   ", None),
        (None, None),
        ("0", 0.0),
    ],
    ids=[
        "plain", "thousands-sep", "currency-symbol",
        "accounting-negative", "negative-with-symbol",
        "empty-string", "whitespace-only", "none", "zero",
    ],
)
def test_parse_amount(raw, expected):
    assert parse_amount(raw) == expected
```

Output:

```
test_parse_amount[plain]                 PASSED
test_parse_amount[thousands-sep]         PASSED
test_parse_amount[accounting-negative]   PASSED
...
```

**Why `ids` matter:** without them pytest generates `test_parse_amount[raw2-expected2]`. With them, a CI failure email tells you *exactly* which business rule broke.

**The `"0"` case:** if someone had written `if not s: return None` *after* converting to float, `0.0` would become `None` and your revenue would be wrong. Always test zero explicitly — falsy values are where bugs hide.
</details>

### Stacking parametrize

Two decorators produce the cartesian product:

```python
@pytest.mark.parametrize("currency", ["USD", "EUR"])
@pytest.mark.parametrize("amount", [0, 100, -50])
def test_convert(currency, amount):
    ...   # runs 6 times
```

Useful, but it explodes fast. Three stacked decorators of 5 values each is 125 tests.

---

## Level 3 — Testing failure paths

### Concept

Half of ETL correctness is *failing loudly on bad data* instead of silently writing garbage. `pytest.raises` asserts that a block raises.

### Exercise 3

Extend `parse_amount` so that unparseable input raises `ValueError` with a message containing the offending value:

```python
raise ValueError(f"Cannot parse amount: {value!r}")
```

Then write tests that:

1. `"abc"` raises `ValueError`.
2. The error message includes the original input (so an on-call engineer can find the bad row).
3. `"1.2.3"` also raises.
4. A test that *fails correctly* if the function starts silently returning `None` instead of raising.

<details>
<summary><b>Solution 3</b></summary>

Implementation change:

```python
    try:
        result = float(s)
    except ValueError:
        raise ValueError(f"Cannot parse amount: {value!r}") from None
    return -result if negative else result
```

Tests:

```python
import pytest
from etl.parsing import parse_amount


def test_garbage_raises():
    with pytest.raises(ValueError):
        parse_amount("abc")


def test_error_message_includes_input():
    with pytest.raises(ValueError, match=r"Cannot parse amount: 'abc'"):
        parse_amount("abc")


@pytest.mark.parametrize("bad", ["abc", "1.2.3", "12-34", "N/A", "--5"])
def test_various_garbage_raises(bad):
    with pytest.raises(ValueError):
        parse_amount(bad)


def test_exception_details_are_accessible():
    with pytest.raises(ValueError) as exc_info:
        parse_amount("N/A")
    assert "N/A" in str(exc_info.value)
```

**Gotchas:**

- `match=` is a **regex search**, not an equality check. Escape special characters: `match=r"\$100 is invalid"`.
- Keep the `with` block to exactly one line. If you put two statements in there and the first raises, the test passes for the wrong reason.
- `pytest.raises(Exception)` is almost always too broad — it will happily pass on a `TypeError` caused by your own typo in the test.

**Real-world design question this raises:** should bad rows raise, or be routed to a quarantine table? Both are valid. Test whichever you chose. If you quarantine, assert the row lands in the reject output *and* that the good rows still process.
</details>

---

## Level 4 — Money, floats, and `pytest.approx`

### Concept

`0.1 + 0.2 == 0.3` is `False` in Python. If you assert exact float equality on computed values, you'll get flaky tests. `pytest.approx` compares with tolerance.

But for money, the better answer is usually to not use floats at all.

### Code under test

```python
# etl/transforms.py
from decimal import Decimal, ROUND_HALF_UP


def line_total_float(unit_price, qty, tax_rate):
    return unit_price * qty * (1 + tax_rate)


def line_total_money(unit_price, qty, tax_rate):
    """Decimal arithmetic, rounded to cents, banker-safe."""
    subtotal = Decimal(str(unit_price)) * Decimal(qty)
    total = subtotal * (Decimal("1") + Decimal(str(tax_rate)))
    return total.quantize(Decimal("0.01"), rounding=ROUND_HALF_UP)
```

### Exercise 4

1. Write a test proving `line_total_float(0.1, 3, 0.0)` is *not* exactly `0.3`.
2. Rewrite it with `pytest.approx` so it passes.
3. Test `line_total_money` with exact `Decimal` equality — no approx needed.
4. Test the half-cent rounding boundary: unit price `1.005`, qty `1`, tax `0` should give `1.01` (not `1.00`, which is what naive `round()` gives).

<details>
<summary><b>Solution 4</b></summary>

```python
from decimal import Decimal
import pytest
from etl.transforms import line_total_float, line_total_money


def test_float_math_is_not_exact():
    assert line_total_float(0.1, 3, 0.0) != 0.3          # demonstrates the trap


def test_float_math_with_approx():
    assert line_total_float(0.1, 3, 0.0) == pytest.approx(0.3)


def test_approx_with_explicit_tolerance():
    # within half a cent
    assert line_total_float(19.99, 3, 0.08) == pytest.approx(64.7676, abs=0.005)


def test_decimal_is_exact():
    assert line_total_money(19.99, 3, Decimal("0.08")) == Decimal("64.77")


def test_half_cent_rounds_up():
    assert line_total_money("1.005", 1, 0) == Decimal("1.01")


def test_zero_quantity():
    assert line_total_money(19.99, 0, 0.08) == Decimal("0.00")
```

**`approx` cheat sheet:**

```python
pytest.approx(0.3)                    # rel tolerance 1e-6, the default
pytest.approx(0.3, abs=0.01)          # absolute tolerance
pytest.approx(0.3, rel=0.01)          # 1% relative
pytest.approx([1.0, 2.0])             # works on lists
pytest.approx({"a": 1.0})             # and dicts
```

**Why `Decimal(str(unit_price))` and not `Decimal(unit_price)`:** `Decimal(0.1)` gives you `0.1000000000000000055511151231257827` — it faithfully reproduces the float's error. Going through `str` gives you `Decimal("0.1")`.

**Real pipeline lesson:** if your warehouse column is `NUMERIC(18,2)` and your Python is float, you will eventually be off by a cent on a reconciliation report and spend a day finding it. Test the rounding boundary (`.005`, `.015`, `.025`) explicitly — `round()` in Python uses banker's rounding and gives `1.0` for `round(1.005, 2)`, which finance will not accept.
</details>

---

## Level 5 — Fixtures

### Concept

A fixture is setup code that tests request **by naming it as a parameter**. pytest matches the parameter name to a fixture function and passes the return value in.

```python
@pytest.fixture
def sample():
    return {"a": 1}

def test_thing(sample):     # pytest sees "sample", calls the fixture, injects result
    assert sample["a"] == 1
```

Fixtures using `yield` get teardown for free:

```python
@pytest.fixture
def conn():
    c = connect()
    yield c          # everything before = setup
    c.close()        # everything after = teardown, runs even if the test fails
```

### Code under test

```python
# etl/transforms.py
import pandas as pd

CUSTOMER_COLUMNS = ["customer_id", "email", "signup_date", "country"]


def clean_customers(df: pd.DataFrame) -> pd.DataFrame:
    """Normalize customer records. Must be safe on empty input."""
    df = df.copy()
    df["email"] = df["email"].astype("string").str.strip().str.lower()
    df["email"] = df["email"].replace("", pd.NA)
    df["country"] = df["country"].astype("string").str.strip().str.upper()
    df = df[df["customer_id"].notna()]
    df = df.drop_duplicates(subset=["customer_id"], keep="last")
    return df[CUSTOMER_COLUMNS].reset_index(drop=True)
```

### Exercise 5

Build a `raw_customers` fixture containing these deliberately messy rows:

```
c1 |  BOB@X.COM   | 2024-01-01 | usa
c1 | bob@x.com    | 2024-01-05 | USA      <- duplicate id, later record wins
c2 | "   "        | 2024-02-01 | uk       <- blank email
c3 | eve@x.com    | 2024-03-01 |  gb      <- country needs trimming
None| mal@x.com   | 2024-04-01 | de       <- null primary key
```

Then write tests for: email lowercasing, blank email → NA, dedup keeping the last record, null-PK row dropped, country uppercased and trimmed.

**Then the important one:** add a separate `empty_customers` fixture (zero rows, correct columns) and assert `clean_customers` returns an empty frame with all the expected columns instead of crashing.

<details>
<summary><b>Solution 5</b></summary>

```python
import pandas as pd
import pytest
from etl.transforms import clean_customers, CUSTOMER_COLUMNS


@pytest.fixture
def raw_customers():
    return pd.DataFrame(
        {
            "customer_id": ["c1", "c1", "c2", "c3", None],
            "email": ["  BOB@X.COM ", "bob@x.com", "   ", "eve@x.com", "mal@x.com"],
            "signup_date": ["2024-01-01", "2024-01-05", "2024-02-01",
                            "2024-03-01", "2024-04-01"],
            "country": ["usa", "USA", "uk", " gb ", "de"],
        }
    )


@pytest.fixture
def empty_customers():
    return pd.DataFrame({c: pd.Series(dtype="object") for c in CUSTOMER_COLUMNS})


def test_email_is_lowercased(raw_customers):
    out = clean_customers(raw_customers)
    assert out.loc[out.customer_id == "c1", "email"].iloc[0] == "bob@x.com"


def test_blank_email_becomes_na(raw_customers):
    out = clean_customers(raw_customers)
    assert pd.isna(out.loc[out.customer_id == "c2", "email"].iloc[0])


def test_duplicate_ids_removed_keeping_last(raw_customers):
    out = clean_customers(raw_customers)
    assert out["customer_id"].is_unique
    assert out.loc[out.customer_id == "c1", "signup_date"].iloc[0] == "2024-01-05"


def test_null_primary_key_dropped(raw_customers):
    out = clean_customers(raw_customers)
    assert out["customer_id"].notna().all()
    assert len(out) == 3


def test_country_normalized(raw_customers):
    out = clean_customers(raw_customers)
    assert set(out["country"]) == {"USA", "UK", "GB"}


def test_empty_input_returns_empty_with_schema(empty_customers):
    out = clean_customers(empty_customers)
    assert len(out) == 0
    assert list(out.columns) == CUSTOMER_COLUMNS


def test_output_index_is_clean(raw_customers):
    out = clean_customers(raw_customers)
    assert list(out.index) == list(range(len(out)))
```

**Why the empty-frame test earns its keep:** every incremental pipeline eventually gets a batch with zero new rows — a holiday, an upstream outage, a filter that matched nothing. If your transform crashes on empty input, your pipeline fails at 3am on Christmas. This is the single most under-tested case in ETL.

**Why `reset_index(drop=True)`:** after filtering, pandas keeps the original index. Write that to Parquet and you get a stray `__index_level_0__` column in your warehouse.

**Fixture scoping note:** `raw_customers` is function-scoped (the default), so each test gets a *fresh* DataFrame. That matters because `clean_customers` could mutate it. If you made this `scope="module"`, a mutation in one test would corrupt the next — a classic source of "passes alone, fails in the suite."
</details>

---

## Level 6 — Factory fixtures

### Concept

Fixtures that return a **function** instead of a value. This is the single highest-leverage pattern in data testing.

The problem it solves: you want 20 tests each with *one* field different. Writing 20 full DataFrames is unreadable and unmaintainable. Instead you write a builder with sensible defaults and override only what the test is about.

### Exercise 6

Write a `make_order` factory fixture producing a single-row order dict with defaults, overridable by keyword. Then write a `make_orders_df` fixture that turns a list of overrides into a DataFrame.

Use them to test a `flag_suspicious_orders` function that flags an order when:
- `amount > 10_000`, **or**
- `billing_country != shipping_country`, **or**
- `qty > 100`.

Write one test per rule, plus a test that a normal order is *not* flagged, plus a test that a row triggering two rules is flagged once (not duplicated).

<details>
<summary><b>Solution 6</b></summary>

Implementation:

```python
# etl/transforms.py
def flag_suspicious_orders(df: pd.DataFrame) -> pd.DataFrame:
    df = df.copy()
    df["is_suspicious"] = (
        (df["amount"] > 10_000)
        | (df["billing_country"] != df["shipping_country"])
        | (df["qty"] > 100)
    )
    return df
```

Fixtures and tests:

```python
import pandas as pd
import pytest
from etl.transforms import flag_suspicious_orders


@pytest.fixture
def make_order():
    def _make(**overrides):
        base = {
            "order_id": "o1",
            "customer_id": "c1",
            "amount": 99.99,
            "qty": 2,
            "billing_country": "US",
            "shipping_country": "US",
            "order_ts": "2024-06-01T12:00:00Z",
        }
        base.update(overrides)
        return base
    return _make


@pytest.fixture
def make_orders_df(make_order):
    def _make(*override_dicts):
        if not override_dicts:
            override_dicts = ({},)
        return pd.DataFrame([make_order(**o) for o in override_dicts])
    return _make


def test_normal_order_not_flagged(make_orders_df):
    out = flag_suspicious_orders(make_orders_df({}))
    assert out["is_suspicious"].iloc[0] is False or not out["is_suspicious"].iloc[0]


def test_high_amount_flagged(make_orders_df):
    out = flag_suspicious_orders(make_orders_df({"amount": 10_000.01}))
    assert out["is_suspicious"].iloc[0]


def test_amount_at_threshold_not_flagged(make_orders_df):
    """Boundary: exactly 10000 is fine, only *above* is suspicious."""
    out = flag_suspicious_orders(make_orders_df({"amount": 10_000}))
    assert not out["is_suspicious"].iloc[0]


def test_country_mismatch_flagged(make_orders_df):
    out = flag_suspicious_orders(make_orders_df({"shipping_country": "NG"}))
    assert out["is_suspicious"].iloc[0]


def test_high_qty_flagged(make_orders_df):
    out = flag_suspicious_orders(make_orders_df({"qty": 101}))
    assert out["is_suspicious"].iloc[0]


def test_multiple_rules_flag_once(make_orders_df):
    out = flag_suspicious_orders(
        make_orders_df({"amount": 50_000, "qty": 500, "shipping_country": "NG"})
    )
    assert len(out) == 1
    assert out["is_suspicious"].iloc[0]


def test_mixed_batch(make_orders_df):
    df = make_orders_df(
        {"order_id": "o1"},
        {"order_id": "o2", "amount": 20_000},
        {"order_id": "o3", "qty": 5},
    )
    out = flag_suspicious_orders(df)
    flagged = set(out.loc[out.is_suspicious, "order_id"])
    assert flagged == {"o2"}
```

**Read `test_high_amount_flagged` again.** The entire test is one line, and that line names exactly the thing being tested. Six months from now you'll still understand it. Compare to a version with a hardcoded 8-column DataFrame where you have to squint to spot which value is the interesting one.

**Note on the boundary test:** `test_amount_at_threshold_not_flagged` is the one that catches a `>` vs `>=` mistake. Every threshold in your pipeline deserves two tests: just below and exactly at.

**Note on fixtures depending on fixtures:** `make_orders_df` requests `make_order`. pytest resolves the whole chain automatically. This composes to arbitrary depth.
</details>

---

## Level 7 — File ingestion with `tmp_path`

### Concept

`tmp_path` is a **built-in fixture** — you don't define it. Request it and pytest hands you a fresh empty `pathlib.Path` directory, unique per test, cleaned up automatically. This is how you test file readers without polluting your repo or fighting over shared fixture files.

Related built-ins: `tmp_path_factory` (session-scoped version), `capsys` (capture stdout), `caplog` (capture logging).

### Code under test

```python
# etl/ingest.py
import pandas as pd

ORDER_DTYPES = {"order_id": "string", "customer_id": "string", "sku": "string"}


def read_orders_csv(path) -> pd.DataFrame:
    df = pd.read_csv(
        path,
        dtype=ORDER_DTYPES,       # keep IDs as strings
        encoding="utf-8-sig",     # strips the Excel BOM
        keep_default_na=False,    # don't turn "NA" country code into null
        na_values=[""],           # but blank really is null
    )
    df.columns = [c.strip().lower().replace(" ", "_") for c in df.columns]
    return df
```

### Exercise 7

Write tests that create CSV files in `tmp_path` and cover these real-world file hazards:

1. **Leading zeros.** `order_id` `"00123"` must stay `"00123"`, not become `123`.
2. **UTF-8 BOM.** A file saved from Excel starts with `\ufeff`. The first column name must not become `"\ufeffordering_id"`.
3. **Embedded commas.** A quoted field `"Smith, John"` is one field, not two.
4. **Embedded newlines.** A quoted address spanning two lines is one row.
5. **The `NA` country code.** Namibia's ISO code is `NA`. pandas turns it into null by default. Assert it survives as the string `"NA"`.
6. **Header-only file.** Zero data rows → empty DataFrame, correct columns, no crash.
7. **Messy headers.** `" Order ID "` becomes `order_id`.
8. **Non-UTF8 bytes.** A latin-1 encoded file raises a clear error rather than producing mojibake silently.

<details>
<summary><b>Solution 7</b></summary>

```python
import pandas as pd
import pytest
from etl.ingest import read_orders_csv


def write(tmp_path, name, content, encoding="utf-8"):
    p = tmp_path / name
    p.write_text(content, encoding=encoding)
    return p


def test_leading_zeros_preserved(tmp_path):
    p = write(tmp_path, "o.csv", "order_id,customer_id,sku,amount\n00123,c1,s1,10\n")
    df = read_orders_csv(p)
    assert df["order_id"].iloc[0] == "00123"


def test_bom_stripped_from_header(tmp_path):
    p = write(tmp_path, "o.csv", "\ufefforder_id,customer_id,sku,amount\n1,c1,s1,10\n")
    df = read_orders_csv(p)
    assert list(df.columns)[0] == "order_id"


def test_quoted_comma_is_one_field(tmp_path):
    p = write(
        tmp_path, "o.csv",
        'order_id,customer_id,sku,notes\n1,c1,s1,"Smith, John"\n',
    )
    df = read_orders_csv(p)
    assert len(df) == 1
    assert df["notes"].iloc[0] == "Smith, John"


def test_quoted_newline_is_one_row(tmp_path):
    p = write(
        tmp_path, "o.csv",
        'order_id,customer_id,sku,address\n1,c1,s1,"12 Main St\nApt 4"\n2,c2,s2,"x"\n',
    )
    df = read_orders_csv(p)
    assert len(df) == 2
    assert "\n" in df["address"].iloc[0]


def test_namibia_country_code_survives(tmp_path):
    p = write(tmp_path, "o.csv", "order_id,customer_id,sku,country\n1,c1,s1,NA\n")
    df = read_orders_csv(p)
    assert df["country"].iloc[0] == "NA"
    assert not pd.isna(df["country"].iloc[0])


def test_header_only_file(tmp_path):
    p = write(tmp_path, "o.csv", "order_id,customer_id,sku,amount\n")
    df = read_orders_csv(p)
    assert len(df) == 0
    assert list(df.columns) == ["order_id", "customer_id", "sku", "amount"]


def test_messy_headers_normalized(tmp_path):
    p = write(tmp_path, "o.csv", " Order ID , Customer ID ,SKU\n1,c1,s1\n")
    df = read_orders_csv(p)
    assert list(df.columns) == ["order_id", "customer_id", "sku"]


def test_blank_field_is_null(tmp_path):
    p = write(tmp_path, "o.csv", "order_id,customer_id,sku,country\n1,c1,s1,\n")
    df = read_orders_csv(p)
    assert pd.isna(df["country"].iloc[0])


def test_bad_encoding_raises(tmp_path):
    p = tmp_path / "o.csv"
    p.write_bytes("order_id,name\n1,Jos\xe9\n".encode("latin-1"))
    with pytest.raises(UnicodeDecodeError):
        read_orders_csv(p)


def test_completely_empty_file_raises(tmp_path):
    p = write(tmp_path, "o.csv", "")
    with pytest.raises(pd.errors.EmptyDataError):
        read_orders_csv(p)
```

**The `dtype` lesson (#1) is worth its own paragraph.** Type inference is the number-one source of silent data corruption in CSV ingestion. Zip code `01234` becomes `1234`. Phone `+1-555` becomes a string in one file and NaN in another. Account number `00000012345678` becomes `12345678.0` in scientific notation. **Always declare dtypes for ID columns.** Never let inference decide.

**The `NA` lesson (#5)** generalizes: pandas' default null list includes `"NA"`, `"N/A"`, `"NULL"`, `"None"`, `"nan"`, `"-"`, and more. If your source system uses any of those as *real values*, set `keep_default_na=False` and declare `na_values` explicitly.

**On `test_bad_encoding_raises`:** you might prefer to *handle* latin-1 rather than raise. Either is fine — but decide, and encode the decision in a test. What you must not do is `errors="ignore"`, which silently drops characters and turns `José` into `Jos`.
</details>

---

## Level 8 — Config and environment with `monkeypatch`

### Concept

`monkeypatch` is a built-in fixture that temporarily changes environment variables, attributes, dict entries, or the working directory — and **undoes it automatically** after the test. That automatic undo is why you use it instead of `os.environ[...] = ...`, which leaks into other tests.

```python
monkeypatch.setenv("KEY", "value")
monkeypatch.delenv("KEY", raising=False)
monkeypatch.setattr("module.function", replacement)
monkeypatch.setattr(obj, "attribute", value)
monkeypatch.setitem(some_dict, "key", value)
monkeypatch.chdir(tmp_path)
```

### Code under test

```python
# etl/config.py
import os
from dataclasses import dataclass


@dataclass(frozen=True)
class Config:
    warehouse_uri: str
    batch_size: int
    env: str
    dry_run: bool


def _as_bool(value: str) -> bool:
    return value.strip().lower() in {"1", "true", "yes", "y", "on"}


def load_config() -> Config:
    uri = os.environ.get("WAREHOUSE_URI")
    if not uri:
        raise RuntimeError("WAREHOUSE_URI is required")

    env = os.environ.get("ETL_ENV", "dev")
    if env not in {"dev", "staging", "prod"}:
        raise ValueError(f"Unknown ETL_ENV: {env!r}")

    return Config(
        warehouse_uri=uri,
        batch_size=int(os.environ.get("BATCH_SIZE", "1000")),
        env=env,
        dry_run=_as_bool(os.environ.get("DRY_RUN", "false")),
    )
```

### Exercise 8

1. Test that a missing `WAREHOUSE_URI` raises with a helpful message.
2. Test defaults apply when optional vars are absent.
3. Parametrize the boolean parsing: `"true"`, `"TRUE"`, `"1"`, `"yes"`, `"on"` → `True`; `"false"`, `"0"`, `"no"`, `""` → `False`.
4. Test an invalid `ETL_ENV` raises.
5. Test that `BATCH_SIZE=abc` raises rather than silently defaulting.
6. **Prove isolation:** write a test that sets an env var, and a second test that asserts it's gone.

<details>
<summary><b>Solution 8</b></summary>

```python
import os
import pytest
from etl.config import load_config


@pytest.fixture
def clean_env(monkeypatch):
    """Remove all ETL vars so tests don't inherit the developer's shell."""
    for var in ["WAREHOUSE_URI", "ETL_ENV", "BATCH_SIZE", "DRY_RUN"]:
        monkeypatch.delenv(var, raising=False)


def test_missing_uri_raises(clean_env):
    with pytest.raises(RuntimeError, match="WAREHOUSE_URI is required"):
        load_config()


def test_defaults_applied(clean_env, monkeypatch):
    monkeypatch.setenv("WAREHOUSE_URI", "duckdb:///tmp/w.db")
    cfg = load_config()
    assert cfg.batch_size == 1000
    assert cfg.env == "dev"
    assert cfg.dry_run is False


@pytest.mark.parametrize(
    "raw,expected",
    [("true", True), ("TRUE", True), ("True", True), ("1", True),
     ("yes", True), ("on", True), (" y ", True),
     ("false", False), ("0", False), ("no", False), ("", False),
     ("banana", False)],
)
def test_dry_run_parsing(clean_env, monkeypatch, raw, expected):
    monkeypatch.setenv("WAREHOUSE_URI", "duckdb:///x")
    monkeypatch.setenv("DRY_RUN", raw)
    assert load_config().dry_run is expected


def test_invalid_env_raises(clean_env, monkeypatch):
    monkeypatch.setenv("WAREHOUSE_URI", "duckdb:///x")
    monkeypatch.setenv("ETL_ENV", "production")     # typo for "prod"
    with pytest.raises(ValueError, match="Unknown ETL_ENV"):
        load_config()


def test_non_numeric_batch_size_raises(clean_env, monkeypatch):
    monkeypatch.setenv("WAREHOUSE_URI", "duckdb:///x")
    monkeypatch.setenv("BATCH_SIZE", "abc")
    with pytest.raises(ValueError):
        load_config()


# --- isolation proof: these two must be run together, in order ---

def test_sets_a_var(monkeypatch):
    monkeypatch.setenv("SCRATCH_VAR", "hello")
    assert os.environ["SCRATCH_VAR"] == "hello"


def test_var_is_gone_afterwards():
    assert "SCRATCH_VAR" not in os.environ
```

**The `clean_env` fixture is not optional in real projects.** Without it, a test passes on your laptop (where `ETL_ENV=dev` is in your `.bashrc`) and fails in CI, or worse: passes in CI and your test never actually exercised the default path. Explicitly clearing the environment makes tests deterministic.

**Note `assert ... is expected` for booleans.** `assert _as_bool("banana") == False` also passes if the function returns `0` or `[]`. `is False` is stricter and catches type drift.

**The `"banana" -> False` case** documents a real design decision: unknown values are treated as false rather than raising. That might be wrong for your project — maybe an unparseable `DRY_RUN` should crash rather than default to running for real against prod. The test is where that decision becomes visible and reviewable.
</details>

---

## Level 9 — Mocking an API source

### Concept

Your extractor calls a paginated REST API with retries. You cannot hit the real API in tests: it's slow, rate-limited, requires secrets, and returns different data every day.

You replace it with a **mock** — an object that records how it was called and returns whatever you tell it to.

```python
from unittest.mock import Mock, call

m = Mock()
m.get.return_value = "hi"
m.get("a", timeout=5)

m.get.assert_called_once_with("a", timeout=5)
assert m.get.call_count == 1

m.get.side_effect = [1, 2, 3]        # successive calls return these
m.get.side_effect = ValueError("x")  # or raise
```

`caplog` is a built-in fixture that captures log records so you can assert on them.

### Code under test

```python
# etl/extract.py
import logging
import time

log = logging.getLogger(__name__)

MAX_RETRIES = 3
PAGE_SIZE = 100


def fetch_all_orders(session, base_url, since):
    """Paginate through the orders API with retry on 429/5xx."""
    results, page = [], 1

    while True:
        payload = _fetch_page_with_retry(session, base_url, since, page)
        rows = payload.get("data", [])
        results.extend(rows)

        if not payload.get("has_more"):
            break
        page += 1
        if page > 1000:
            raise RuntimeError("Pagination did not terminate after 1000 pages")

    return results


def _fetch_page_with_retry(session, base_url, since, page):
    for attempt in range(1, MAX_RETRIES + 1):
        response = session.get(
            base_url,
            params={"since": since, "page": page, "limit": PAGE_SIZE},
            timeout=30,
        )
        if response.status_code == 200:
            return response.json()

        if response.status_code == 429 or response.status_code >= 500:
            wait = 2 ** attempt
            log.warning("Retryable status %s on page %s, sleeping %ss",
                        response.status_code, page, wait)
            time.sleep(wait)
            continue

        raise RuntimeError(f"API error {response.status_code} on page {page}")

    raise RuntimeError(f"Gave up on page {page} after {MAX_RETRIES} attempts")
```

### Exercise 9

Write tests covering:

1. Single page, `has_more: false` → returns those rows, exactly one HTTP call.
2. Three pages → rows concatenated in order, three calls, page numbers 1/2/3.
3. A 429 followed by a 200 → succeeds, two calls made.
4. Persistent 500 → raises after exactly `MAX_RETRIES` attempts.
5. A 404 → raises **immediately**, no retries (it's not retryable).
6. Empty result set (`data: []`, `has_more: false`) → returns `[]`, doesn't crash.
7. A warning is logged on retry.
8. **Make the whole suite fast** by patching `time.sleep` — assert your tests don't actually sleep 14 seconds.

<details>
<summary><b>Solution 9</b></summary>

```python
import logging
from unittest.mock import Mock, call
import pytest
from etl import extract
from etl.extract import fetch_all_orders, MAX_RETRIES


@pytest.fixture(autouse=True)
def no_sleep(monkeypatch):
    """Patch out backoff so the suite runs in milliseconds, and record waits."""
    calls = []
    monkeypatch.setattr(extract.time, "sleep", lambda s: calls.append(s))
    return calls


def make_response(status=200, data=None, has_more=False):
    r = Mock()
    r.status_code = status
    r.json.return_value = {"data": data or [], "has_more": has_more}
    return r


@pytest.fixture
def session():
    return Mock()


def test_single_page(session):
    session.get.return_value = make_response(data=[{"id": 1}, {"id": 2}])
    rows = fetch_all_orders(session, "http://api/orders", "2024-01-01")
    assert rows == [{"id": 1}, {"id": 2}]
    assert session.get.call_count == 1


def test_pagination_concatenates_in_order(session):
    session.get.side_effect = [
        make_response(data=[{"id": 1}], has_more=True),
        make_response(data=[{"id": 2}], has_more=True),
        make_response(data=[{"id": 3}], has_more=False),
    ]
    rows = fetch_all_orders(session, "http://api/orders", "2024-01-01")
    assert [r["id"] for r in rows] == [1, 2, 3]
    assert session.get.call_count == 3
    pages = [c.kwargs["params"]["page"] for c in session.get.call_args_list]
    assert pages == [1, 2, 3]


def test_rate_limit_then_success(session, no_sleep):
    session.get.side_effect = [
        make_response(status=429),
        make_response(data=[{"id": 1}]),
    ]
    rows = fetch_all_orders(session, "http://api/orders", "2024-01-01")
    assert rows == [{"id": 1}]
    assert session.get.call_count == 2
    assert no_sleep == [2]                      # backed off once, 2 seconds


def test_persistent_server_error_gives_up(session, no_sleep):
    session.get.return_value = make_response(status=503)
    with pytest.raises(RuntimeError, match="Gave up on page 1"):
        fetch_all_orders(session, "http://api/orders", "2024-01-01")
    assert session.get.call_count == MAX_RETRIES
    assert no_sleep == [2, 4, 8]                # exponential backoff


def test_client_error_not_retried(session, no_sleep):
    session.get.return_value = make_response(status=404)
    with pytest.raises(RuntimeError, match="API error 404"):
        fetch_all_orders(session, "http://api/orders", "2024-01-01")
    assert session.get.call_count == 1
    assert no_sleep == []                       # never slept


def test_empty_result_set(session):
    session.get.return_value = make_response(data=[], has_more=False)
    assert fetch_all_orders(session, "http://api/orders", "2024-01-01") == []


def test_retry_is_logged(session, caplog):
    session.get.side_effect = [make_response(status=429), make_response(data=[])]
    with caplog.at_level(logging.WARNING):
        fetch_all_orders(session, "http://api/orders", "2024-01-01")
    assert any("Retryable status 429" in r.message % r.args if r.args else
               "Retryable status" in r.message for r in caplog.records)


def test_request_params_are_correct(session):
    session.get.return_value = make_response(data=[])
    fetch_all_orders(session, "http://api/orders", "2024-01-01")
    session.get.assert_called_once_with(
        "http://api/orders",
        params={"since": "2024-01-01", "page": 1, "limit": 100},
        timeout=30,
    )
```

**`autouse=True`** makes `no_sleep` apply to every test in the file without being requested. Use it sparingly — hidden magic — but "never really sleep in tests" is a legitimate case.

**Why assert on `no_sleep == [2, 4, 8]`:** it proves the backoff is exponential and not a constant 1 second, without the test taking 14 seconds to run. Testing *timing behaviour* without *spending time* is the whole trick.

**Why `test_client_error_not_retried` matters:** retrying a 404 or a 401 wastes three minutes per run and does nothing. Retrying a 400 caused by malformed input can, on some APIs, create duplicate records. The retryable/non-retryable split is a real correctness boundary.

**`test_request_params_are_correct` is a contract test.** If someone changes `limit` to 10000 and the API silently caps at 1000, your extract quietly loses rows. This test freezes the contract.

**Mock hygiene:** `assert_called_once_with` is strict about *everything* — args, kwargs, order. That strictness is a feature. If you only care about one argument, inspect `call_args_list` instead of loosening the assertion.
</details>

---

## Level 10 — Databases and idempotency

### Concept

Two rules for database tests:

1. Use a **real database engine**, not a mock. SQLite or DuckDB in-memory starts in milliseconds. Mocking a database means you test your mock, not your SQL.
2. Each test gets a **clean database**. Use a fixture with teardown.

The most important property to test in ETL: **idempotency**. Running the same load twice must produce the same result as running it once. Pipelines get retried — by Airflow, by the on-call engineer, by a network blip. A non-idempotent load doubles your revenue numbers.

### Code under test

```python
# etl/load.py
import duckdb

DDL = """
CREATE TABLE IF NOT EXISTS customers (
    customer_id  VARCHAR PRIMARY KEY,
    email        VARCHAR,
    country      VARCHAR,
    updated_at   TIMESTAMP
);
"""


def init_schema(conn):
    conn.execute(DDL)


def upsert_customers(conn, rows):
    """Merge rows on customer_id. Later updated_at wins."""
    if not rows:
        return 0

    conn.execute("BEGIN TRANSACTION")
    try:
        conn.executemany(
            """
            INSERT INTO customers (customer_id, email, country, updated_at)
            VALUES (?, ?, ?, ?)
            ON CONFLICT (customer_id) DO UPDATE SET
                email      = excluded.email,
                country    = excluded.country,
                updated_at = excluded.updated_at
            WHERE excluded.updated_at >= customers.updated_at
            """,
            [(r["customer_id"], r["email"], r["country"], r["updated_at"])
             for r in rows],
        )
        conn.execute("COMMIT")
    except Exception:
        conn.execute("ROLLBACK")
        raise
    return len(rows)
```

### Exercise 10

1. A `conn` fixture giving each test a fresh in-memory DuckDB with the schema applied.
2. Insert two rows → both present.
3. **Idempotency:** run the identical load twice → still two rows, identical values.
4. Update: same `customer_id`, newer `updated_at`, different email → email changes.
5. **Out-of-order arrival:** same `customer_id`, *older* `updated_at` → the newer value is **kept**, not overwritten. (This is the bug that late-arriving CDC events cause.)
6. Empty input list → returns 0, table unchanged, no crash.
7. **Atomicity:** a batch where one row violates a constraint leaves the table completely unchanged — no partial write.

<details>
<summary><b>Solution 10</b></summary>

```python
from datetime import datetime
import duckdb
import pytest
from etl.load import init_schema, upsert_customers


@pytest.fixture
def conn():
    c = duckdb.connect(":memory:")
    init_schema(c)
    yield c
    c.close()


def row(cid="c1", email="a@x.com", country="US", ts="2024-01-01 00:00:00"):
    return {"customer_id": cid, "email": email, "country": country,
            "updated_at": datetime.fromisoformat(ts)}


def fetch_all(conn):
    return conn.execute(
        "SELECT customer_id, email, country, updated_at "
        "FROM customers ORDER BY customer_id"
    ).fetchall()


def test_inserts_rows(conn):
    upsert_customers(conn, [row("c1"), row("c2", email="b@x.com")])
    result = fetch_all(conn)
    assert len(result) == 2
    assert {r[0] for r in result} == {"c1", "c2"}


def test_load_is_idempotent(conn):
    batch = [row("c1"), row("c2", email="b@x.com")]
    upsert_customers(conn, batch)
    first = fetch_all(conn)

    upsert_customers(conn, batch)          # exact same batch again
    second = fetch_all(conn)

    assert first == second
    assert len(second) == 2


def test_newer_record_updates(conn):
    upsert_customers(conn, [row("c1", email="old@x.com", ts="2024-01-01 00:00:00")])
    upsert_customers(conn, [row("c1", email="new@x.com", ts="2024-06-01 00:00:00")])
    result = fetch_all(conn)
    assert len(result) == 1
    assert result[0][1] == "new@x.com"


def test_older_record_does_not_overwrite(conn):
    """Late-arriving CDC event must not clobber fresher state."""
    upsert_customers(conn, [row("c1", email="new@x.com", ts="2024-06-01 00:00:00")])
    upsert_customers(conn, [row("c1", email="stale@x.com", ts="2024-01-01 00:00:00")])
    result = fetch_all(conn)
    assert result[0][1] == "new@x.com"


def test_equal_timestamp_applies_update(conn):
    """Boundary: >= means a replay of the same timestamp is allowed."""
    upsert_customers(conn, [row("c1", email="a@x.com", ts="2024-06-01 00:00:00")])
    upsert_customers(conn, [row("c1", email="b@x.com", ts="2024-06-01 00:00:00")])
    assert fetch_all(conn)[0][1] == "b@x.com"


def test_empty_batch_is_noop(conn):
    upsert_customers(conn, [row("c1")])
    before = fetch_all(conn)
    assert upsert_customers(conn, []) == 0
    assert fetch_all(conn) == before


def test_batch_is_atomic(conn):
    upsert_customers(conn, [row("c1", email="good@x.com")])
    before = fetch_all(conn)

    bad_batch = [
        row("c2", email="ok@x.com"),
        {"customer_id": None, "email": "x", "country": "US",
         "updated_at": datetime(2024, 1, 1)},          # violates PK NOT NULL
    ]
    with pytest.raises(Exception):
        upsert_customers(conn, bad_batch)

    assert fetch_all(conn) == before        # c2 must NOT be there
```

**`test_load_is_idempotent` is the single most valuable test in this entire guide.** Write it for every load function you ever build. Airflow will retry your task. The retry must not duplicate rows.

**`test_older_record_does_not_overwrite`** encodes a rule that bites everyone doing CDC or Kafka ingestion: **events do not arrive in order**. Without the `WHERE excluded.updated_at >= customers.updated_at` guard, a replayed old event silently reverts a customer's email to a value from six months ago. There's no error, no alert — just wrong data.

**`test_equal_timestamp_applies_update`** documents the `>=` vs `>` choice. With `>`, an exact replay is a no-op. With `>=`, it re-applies. Both defensible; the test makes the choice explicit so a future reviewer sees it.

**`test_batch_is_atomic`** is the difference between "the pipeline failed" and "the pipeline failed and left half a batch in the warehouse." The second one is far worse because a naive retry then double-loads the first half.

**On SQLite vs DuckDB:** SQLite is in the standard library (`sqlite3`, zero install) but is loosely typed and lacks many analytical SQL features. DuckDB is closer to Snowflake/BigQuery semantics — window functions, `QUALIFY`, proper types — so tests transfer better. Use DuckDB when your production SQL is analytical.

**What this doesn't cover:** dialect-specific SQL. If your production query uses Snowflake's `MERGE ... WHEN MATCHED AND`, DuckDB may not parse it identically. For those, keep a small set of integration tests marked `@pytest.mark.integration` that run against a real dev warehouse in CI only.
</details>

---

## Level 11 — Spark: session fixture and schema contracts

### Concept

Two things make Spark tests bearable:

1. A **session-scoped** SparkSession fixture in `conftest.py`. Starting Spark costs 5–15 seconds; do it once.
2. Configuration tuned for tiny data. Default shuffle partitions is 200 — on a 4-row DataFrame that's 200 empty tasks per shuffle, and your suite crawls.

### `tests/conftest.py`

```python
import pytest
from pyspark.sql import SparkSession


@pytest.fixture(scope="session")
def spark():
    spark = (
        SparkSession.builder
        .master("local[2]")
        .appName("etl-tests")
        .config("spark.sql.shuffle.partitions", "1")
        .config("spark.default.parallelism", "1")
        .config("spark.sql.adaptive.enabled", "false")
        .config("spark.ui.enabled", "false")
        .config("spark.sql.session.timeZone", "UTC")
        .config("spark.driver.host", "localhost")
        .getOrCreate()
    )
    spark.sparkContext.setLogLevel("ERROR")
    yield spark
    spark.stop()
```

Pin the session timezone to UTC. Otherwise your tests pass in London and fail in Mumbai, and you will not enjoy finding out why.

### Code under test

```python
# etl/spark_jobs.py
from pyspark.sql import DataFrame, functions as F
from pyspark.sql.types import (StructType, StructField, StringType,
                               DecimalType, TimestampType, IntegerType)

ORDERS_SCHEMA = StructType([
    StructField("order_id",    StringType(),        nullable=False),
    StructField("customer_id", StringType(),        nullable=False),
    StructField("amount",      DecimalType(18, 2),  nullable=True),
    StructField("qty",         IntegerType(),       nullable=True),
    StructField("order_ts",    TimestampType(),     nullable=True),
])


class SchemaError(Exception):
    pass


def validate_schema(df: DataFrame, expected: StructType) -> None:
    """Fail loudly on schema drift: missing columns or wrong types."""
    actual = {f.name: f.dataType for f in df.schema.fields}

    missing = [f.name for f in expected.fields if f.name not in actual]
    if missing:
        raise SchemaError(f"Missing columns: {sorted(missing)}")

    wrong = [
        f"{f.name}: expected {f.dataType.simpleString()}, "
        f"got {actual[f.name].simpleString()}"
        for f in expected.fields
        if actual[f.name] != f.dataType
    ]
    if wrong:
        raise SchemaError("Type mismatch -> " + "; ".join(wrong))


def select_contract_columns(df: DataFrame, expected: StructType) -> DataFrame:
    """Drop unexpected columns so downstream writes stay stable."""
    validate_schema(df, expected)
    return df.select(*[f.name for f in expected.fields])
```

### Exercise 11

1. A conforming DataFrame passes validation.
2. A missing column raises `SchemaError` naming that column.
3. A wrong type (`amount` as `string`) raises, and the message says both expected and actual.
4. An **extra** column passes validation but is dropped by `select_contract_columns`. (New upstream columns shouldn't break you — but shouldn't leak into the warehouse either.)
5. Column **order** doesn't matter for validation, but output order is the contract's order.
6. An empty DataFrame with the correct schema passes.

<details>
<summary><b>Solution 11</b></summary>

```python
from decimal import Decimal
import pytest
from pyspark.sql.types import (StructType, StructField, StringType,
                               DecimalType, IntegerType, TimestampType)
from etl.spark_jobs import (ORDERS_SCHEMA, SchemaError,
                            validate_schema, select_contract_columns)

pytestmark = pytest.mark.spark          # marks EVERY test in this file


@pytest.fixture
def good_orders(spark):
    return spark.createDataFrame([], ORDERS_SCHEMA)


def test_conforming_schema_passes(good_orders):
    validate_schema(good_orders, ORDERS_SCHEMA)     # no exception == pass


def test_empty_dataframe_passes(good_orders):
    assert good_orders.count() == 0
    validate_schema(good_orders, ORDERS_SCHEMA)


def test_missing_column_raises(spark):
    partial = StructType([f for f in ORDERS_SCHEMA.fields if f.name != "qty"])
    df = spark.createDataFrame([], partial)
    with pytest.raises(SchemaError, match=r"Missing columns: \['qty'\]"):
        validate_schema(df, ORDERS_SCHEMA)


def test_wrong_type_raises_with_detail(spark):
    drifted = StructType([
        StructField("order_id", StringType()),
        StructField("customer_id", StringType()),
        StructField("amount", StringType()),          # <- drifted from decimal
        StructField("qty", IntegerType()),
        StructField("order_ts", TimestampType()),
    ])
    df = spark.createDataFrame([], drifted)
    with pytest.raises(SchemaError) as exc:
        validate_schema(df, ORDERS_SCHEMA)
    msg = str(exc.value)
    assert "amount" in msg
    assert "decimal(18,2)" in msg
    assert "string" in msg


def test_extra_column_is_tolerated_then_dropped(spark):
    extended = StructType(
        list(ORDERS_SCHEMA.fields) + [StructField("promo_code", StringType())]
    )
    df = spark.createDataFrame([], extended)

    validate_schema(df, ORDERS_SCHEMA)                # tolerated
    out = select_contract_columns(df, ORDERS_SCHEMA)
    assert "promo_code" not in out.columns


def test_column_order_normalized(spark):
    shuffled = StructType([
        StructField("qty", IntegerType()),
        StructField("order_ts", TimestampType()),
        StructField("order_id", StringType()),
        StructField("amount", DecimalType(18, 2)),
        StructField("customer_id", StringType()),
    ])
    df = spark.createDataFrame([], shuffled)
    out = select_contract_columns(df, ORDERS_SCHEMA)
    assert out.columns == [f.name for f in ORDERS_SCHEMA.fields]


def test_decimal_precision_drift_is_caught(spark):
    """decimal(18,2) vs decimal(10,2) is real drift — money can overflow."""
    drifted = StructType([
        StructField("order_id", StringType()),
        StructField("customer_id", StringType()),
        StructField("amount", DecimalType(10, 2)),
        StructField("qty", IntegerType()),
        StructField("order_ts", TimestampType()),
    ])
    df = spark.createDataFrame([], drifted)
    with pytest.raises(SchemaError, match="amount"):
        validate_schema(df, ORDERS_SCHEMA)
```

**`pytestmark = pytest.mark.spark`** at module level applies the marker to every test in the file. Now `pytest -m "not spark"` gives you a sub-second feedback loop while you work on parsing logic.

**Why schema tests earn their keep:** upstream teams add, rename, and retype columns without telling you. The failure modes without a contract test:
- New column → your `SELECT *` writes it to the warehouse, the table schema changes, a downstream dashboard breaks.
- Renamed column → `NULL` for everything, silently, forever.
- `int` → `string` → your `SUM()` returns null or errors deep in a nightly job.

A schema test converts all three from "discovered by a business user three weeks later" into "pipeline fails immediately with a clear message."

**Note on `nullable`:** Spark's nullable flag is notoriously unreliable — it changes across joins and reads. Comparing `dataType` (as above) rather than the full `StructField` avoids endless false failures. If you *do* need nullability enforced, assert it separately and only on columns where you genuinely control it.

**Always pass an explicit schema to `createDataFrame` in tests.** Inference on three rows guesses `LongType` for integers and chokes on all-null columns, giving you failures that have nothing to do with your logic.
</details>

---

## Level 12 — Joins, nulls, and fan-out

### Concept

Joins are where ETL row counts go wrong. Three classic bugs, all silent:

1. **Fan-out** — the "dimension" you joined has duplicate keys, so your fact table grows and every metric double-counts.
2. **Null keys** — `NULL = NULL` is false in SQL, so rows with null join keys vanish.
3. **Dirty keys** — `"ABC "` doesn't equal `"abc"`, so rows fall out for invisible reasons.

`chispa` gives you `assert_df_equality` for comparing whole DataFrames readably.

```python
from chispa import assert_df_equality
assert_df_equality(actual, expected, ignore_row_order=True, ignore_nullable=True)
```

### Code under test

```python
# etl/spark_jobs.py
def enrich_orders(orders: DataFrame, customers: DataFrame) -> DataFrame:
    """Left-join customer attributes onto orders without changing row count."""
    dim = (customers
           .withColumn("join_key", F.upper(F.trim(F.col("customer_id"))))
           .dropDuplicates(["join_key"])
           .select("join_key", "country", "segment"))

    return (orders
            .withColumn("join_key", F.upper(F.trim(F.col("customer_id"))))
            .join(dim, on="join_key", how="left")
            .drop("join_key")
            .withColumn("country", F.coalesce(F.col("country"), F.lit("UNKNOWN")))
            .withColumn("segment", F.coalesce(F.col("segment"), F.lit("UNKNOWN"))))
```

### Exercise 12

1. A clean join enriches correctly.
2. **Row count is preserved** even when the customer dimension has duplicate `customer_id`s.
3. An order with no matching customer keeps the order and gets `"UNKNOWN"` — not dropped.
4. Key matching survives whitespace and case differences (`" c1 "` matches `"C1"`).
5. An order with a **null** `customer_id` is still present in the output with `"UNKNOWN"`.
6. Empty orders input → empty output, correct columns, no crash.
7. Empty customers dimension → all orders present, all `"UNKNOWN"`.

<details>
<summary><b>Solution 12</b></summary>

```python
import pytest
from chispa import assert_df_equality
from etl.spark_jobs import enrich_orders

pytestmark = pytest.mark.spark

ORDER_COLS = ["order_id", "customer_id", "amount"]
CUST_COLS = ["customer_id", "country", "segment"]


@pytest.fixture
def make_orders(spark):
    def _make(rows):
        return spark.createDataFrame(rows, ORDER_COLS)
    return _make


@pytest.fixture
def make_customers(spark):
    def _make(rows):
        return spark.createDataFrame(rows, CUST_COLS)
    return _make


def test_basic_enrichment(make_orders, make_customers):
    orders = make_orders([("o1", "c1", 100.0)])
    customers = make_customers([("c1", "US", "retail")])

    out = enrich_orders(orders, customers).collect()
    assert len(out) == 1
    assert out[0].country == "US"
    assert out[0].segment == "retail"


def test_duplicate_dimension_keys_do_not_fan_out(make_orders, make_customers):
    """THE classic bug. 2 orders must stay 2 orders."""
    orders = make_orders([("o1", "c1", 100.0), ("o2", "c1", 200.0)])
    customers = make_customers([
        ("c1", "US", "retail"),
        ("c1", "US", "retail"),      # duplicate row in the dimension
        ("c1", "CA", "wholesale"),   # and a conflicting one
    ])

    out = enrich_orders(orders, customers)
    assert out.count() == 2          # NOT 6


def test_unmatched_order_is_kept(make_orders, make_customers):
    orders = make_orders([("o1", "ghost", 100.0)])
    customers = make_customers([("c1", "US", "retail")])

    out = enrich_orders(orders, customers).collect()
    assert len(out) == 1
    assert out[0].country == "UNKNOWN"
    assert out[0].segment == "UNKNOWN"


@pytest.mark.parametrize("order_key,cust_key", [
    (" c1 ", "c1"),
    ("C1", "c1"),
    ("c1", "  C1  "),
    ("  C1", "c1  "),
])
def test_dirty_keys_still_match(make_orders, make_customers, order_key, cust_key):
    orders = make_orders([("o1", order_key, 100.0)])
    customers = make_customers([(cust_key, "US", "retail")])
    assert enrich_orders(orders, customers).collect()[0].country == "US"


def test_null_customer_id_row_survives(make_orders, make_customers):
    """A null join key must not silently delete the order."""
    orders = make_orders([("o1", None, 100.0), ("o2", "c1", 50.0)])
    customers = make_customers([("c1", "US", "retail")])

    out = enrich_orders(orders, customers)
    assert out.count() == 2
    ghost = out.filter("order_id = 'o1'").collect()[0]
    assert ghost.country == "UNKNOWN"


def test_empty_orders(spark, make_customers):
    orders = spark.createDataFrame([], "order_id string, customer_id string, amount double")
    out = enrich_orders(orders, make_customers([("c1", "US", "retail")]))
    assert out.count() == 0
    assert set(["order_id", "country", "segment"]).issubset(set(out.columns))


def test_empty_customer_dimension(spark, make_orders):
    customers = spark.createDataFrame([], "customer_id string, country string, segment string")
    out = enrich_orders(make_orders([("o1", "c1", 100.0)]), customers)
    assert out.count() == 1
    assert out.collect()[0].country == "UNKNOWN"


def test_exact_output_with_chispa(spark, make_orders, make_customers):
    orders = make_orders([("o1", "c1", 100.0), ("o2", "zz", 50.0)])
    customers = make_customers([("c1", "US", "retail")])

    expected = spark.createDataFrame(
        [("o1", "c1", 100.0, "US", "retail"),
         ("o2", "zz", 50.0, "UNKNOWN", "UNKNOWN")],
        "order_id string, customer_id string, amount double, "
        "country string, segment string",
    )
    actual = enrich_orders(orders, customers).select(*expected.columns)
    assert_df_equality(actual, expected, ignore_row_order=True, ignore_nullable=True)
```

**Add a row-count assertion to every join test you write.** `assert out.count() == orders.count()` for a left join with a deduplicated dimension. If someone later removes the `dropDuplicates`, the test catches it before your revenue report triples.

**Why `ignore_nullable=True` in chispa:** Spark flips nullable flags after joins and coalesce. Without this flag you get failures that are pure noise. Keep the flag on unless nullability is what you're testing.

**Why `ignore_row_order=True`:** Spark makes no ordering guarantee without an explicit `orderBy`. A test that depends on row order is a test that will flake when the partition count changes.

**Null keys in Spark, specifically:** an inner join drops null-key rows entirely; a left join keeps them with nulls on the right. If you genuinely need `NULL` to match `NULL` (rare, but happens with composite keys), use `df1["k"].eqNullSafe(df2["k"])` — and test it, because nobody remembers that operator exists.

**One more real-world variant to try on your own:** a many-to-many join where *both* sides have duplicates. Write the test first, watch the row count explode, then decide whether you want a pre-aggregation or a window-function pick-one. The test makes the design decision concrete.
</details>

---

## Level 13 — Dedup and late-arriving data

### Concept

CDC feeds, Kafka topics, and API replays all deliver the **same key more than once, out of order**. You need "latest version per key" — and you need it to be *deterministic*, because a tie broken arbitrarily gives different results on every run and makes your tests flaky.

### Code under test

```python
# etl/spark_jobs.py
from pyspark.sql import Window


def latest_per_key(df: DataFrame, key: str, order_col: str,
                   tiebreak_col: str = None) -> DataFrame:
    """Keep one row per key: highest order_col, ties broken deterministically."""
    ordering = [F.col(order_col).desc_nulls_last()]
    if tiebreak_col:
        ordering.append(F.col(tiebreak_col).desc_nulls_last())

    w = Window.partitionBy(key).orderBy(*ordering)
    return (df
            .withColumn("_rn", F.row_number().over(w))
            .filter(F.col("_rn") == 1)
            .drop("_rn"))


def drop_late_events(df: DataFrame, ts_col: str, watermark) -> DataFrame:
    """Route events older than the watermark out of the main stream."""
    return df.filter(F.col(ts_col) >= F.lit(watermark).cast("timestamp"))
```

### Exercise 13

1. Three versions of the same key → one row, the one with the newest timestamp.
2. Multiple distinct keys are each reduced independently.
3. **Out-of-order input:** feed the rows newest-first and assert you get the same answer. Order of input must not matter.
4. **Deterministic tie-break:** two rows with the *identical* timestamp → the tiebreak column decides, and the result is the same across repeated runs.
5. A row whose ordering column is **null** loses to any non-null row.
6. A key where *every* row has a null timestamp still yields exactly one row (not zero).
7. `drop_late_events` removes events before the watermark and keeps events exactly at it.

<details>
<summary><b>Solution 13</b></summary>

```python
import pytest
from etl.spark_jobs import latest_per_key, drop_late_events

pytestmark = pytest.mark.spark

SCHEMA = "key string, payload string, event_ts timestamp, ingest_seq int"


@pytest.fixture
def make_events(spark):
    def _make(rows):
        return spark.createDataFrame(rows, SCHEMA)
    return _make


def ts(s):
    from datetime import datetime
    return datetime.fromisoformat(s)


def test_keeps_newest_version(make_events):
    df = make_events([
        ("k1", "v1", ts("2024-01-01T00:00:00"), 1),
        ("k1", "v2", ts("2024-02-01T00:00:00"), 2),
        ("k1", "v3", ts("2024-03-01T00:00:00"), 3),
    ])
    out = latest_per_key(df, "key", "event_ts").collect()
    assert len(out) == 1
    assert out[0].payload == "v3"


def test_keys_are_independent(make_events):
    df = make_events([
        ("k1", "a", ts("2024-01-01T00:00:00"), 1),
        ("k1", "b", ts("2024-02-01T00:00:00"), 2),
        ("k2", "c", ts("2024-01-01T00:00:00"), 3),
    ])
    out = {r.key: r.payload for r in latest_per_key(df, "key", "event_ts").collect()}
    assert out == {"k1": "b", "k2": "c"}


def test_input_order_does_not_matter(make_events):
    rows = [
        ("k1", "old", ts("2024-01-01T00:00:00"), 1),
        ("k1", "new", ts("2024-03-01T00:00:00"), 2),
    ]
    forward = latest_per_key(make_events(rows), "key", "event_ts").collect()
    backward = latest_per_key(make_events(rows[::-1]), "key", "event_ts").collect()
    assert forward[0].payload == backward[0].payload == "new"


def test_tie_broken_deterministically(make_events):
    same_ts = ts("2024-01-01T00:00:00")
    df = make_events([
        ("k1", "first_arrival", same_ts, 1),
        ("k1", "second_arrival", same_ts, 2),
    ])
    results = {
        latest_per_key(df, "key", "event_ts", "ingest_seq").collect()[0].payload
        for _ in range(5)                      # run it five times
    }
    assert results == {"second_arrival"}       # same answer every time


def test_tie_without_tiebreak_is_nondeterministic_by_design(make_events):
    """Documents the hazard: no tiebreak column = arbitrary winner."""
    same_ts = ts("2024-01-01T00:00:00")
    df = make_events([("k1", "a", same_ts, 1), ("k1", "b", same_ts, 2)])
    out = latest_per_key(df, "key", "event_ts").collect()
    assert len(out) == 1                       # exactly one row is all we can assert
    assert out[0].payload in {"a", "b"}


def test_null_timestamp_loses(make_events):
    df = make_events([
        ("k1", "has_ts", ts("2024-01-01T00:00:00"), 1),
        ("k1", "no_ts", None, 2),
    ])
    assert latest_per_key(df, "key", "event_ts").collect()[0].payload == "has_ts"


def test_all_null_timestamps_still_yields_one_row(make_events):
    df = make_events([("k1", "a", None, 1), ("k1", "b", None, 2)])
    out = latest_per_key(df, "key", "event_ts", "ingest_seq").collect()
    assert len(out) == 1
    assert out[0].payload == "b"


def test_empty_input(spark):
    df = spark.createDataFrame([], SCHEMA)
    assert latest_per_key(df, "key", "event_ts").count() == 0


@pytest.mark.parametrize("event_time,kept", [
    ("2024-06-01T00:00:00", True),      # exactly at watermark -> kept
    ("2024-06-01T00:00:01", True),
    ("2024-05-31T23:59:59", False),     # one second late -> dropped
])
def test_watermark_boundary(make_events, event_time, kept):
    df = make_events([("k1", "v", ts(event_time), 1)])
    out = drop_late_events(df, "event_ts", "2024-06-01 00:00:00")
    assert (out.count() == 1) is kept
```

**`test_tie_broken_deterministically` running the transform five times** is the practical way to catch nondeterminism. A single run of a nondeterministic transform passes about half the time — which is exactly the profile of a test that fails randomly in CI six weeks from now.

**Always give `row_number()` a full ordering.** `event_ts` alone is not a total order if timestamps can collide (and with second-precision source systems, they collide constantly). Add an ingestion sequence, an offset, a file name, anything monotonic. Without it, two runs over the same data can produce different warehouse contents.

**`desc_nulls_last()` vs `desc()`:** in Spark, `desc()` puts nulls *first*, so a row with a null timestamp would win. That is almost never what you want and it's invisible until you test it.

**The watermark boundary test** pins down `>=` vs `>`. Off-by-one-second at a watermark means you either drop a legitimate event or reprocess a duplicate. Both show up as a reconciliation discrepancy that takes hours to trace.

**Extension to try:** add a `deleted` flag to the source and make `latest_per_key` route tombstoned keys to a separate DataFrame. Test that a delete arriving *before* an update (out of order) doesn't resurrect the record.
</details>

---

## Level 14 — Slowly Changing Dimension Type 2

### Concept

SCD2 keeps history: when a tracked attribute changes, you close the old row (`valid_to = today`, `is_current = false`) and open a new one. It's the hardest common ETL pattern to get right and the one most worth testing, because errors are invisible until someone runs a point-in-time report.

### Code under test

```python
# etl/scd.py
from pyspark.sql import DataFrame, functions as F

HIGH_DATE = "9999-12-31"


def _row_hash(cols):
    """Hash tracked attributes. NULL and '' must hash differently."""
    parts = [F.coalesce(F.col(c).cast("string"), F.lit("<NULL>")) for c in cols]
    return F.sha2(F.concat_ws("||", *parts), 256)


def apply_scd2(current: DataFrame, incoming: DataFrame, key: str,
               tracked: list, effective_date: str) -> DataFrame:
    tracked = list(tracked)
    out_cols = [key] + tracked + ["valid_from", "valid_to", "is_current"]
    eff = F.lit(effective_date).cast("date")
    high = F.lit(HIGH_DATE).cast("date")

    inc = (incoming
           .select(F.col(key), *[F.col(c).alias(f"n_{c}") for c in tracked])
           .withColumn("n_h", _row_hash([f"n_{c}" for c in tracked])))

    open_rows = (current.filter(F.col("is_current"))
                 .select(F.col(key),
                         *[F.col(c).alias(f"o_{c}") for c in tracked],
                         F.col("valid_from").alias("o_valid_from"))
                 .withColumn("o_h", _row_hash([f"o_{c}" for c in tracked])))

    history = current.filter(~F.col("is_current")).select(*out_cols)
    j = open_rows.join(inc, on=key, how="full_outer")

    def project(prefix, valid_from, valid_to, is_current):
        return [F.col(key)] + \
               [F.col(f"{prefix}{c}").alias(c) for c in tracked] + \
               [valid_from.alias("valid_from"),
                valid_to.alias("valid_to"),
                F.lit(is_current).alias("is_current")]

    unchanged = j.filter(F.col("o_h") == F.col("n_h")).select(
        *project("o_", F.col("o_valid_from"), high, True))

    changed = j.filter(F.col("o_h").isNotNull() & F.col("n_h").isNotNull()
                       & (F.col("o_h") != F.col("n_h")))
    closed_old = changed.select(*project("o_", F.col("o_valid_from"), eff, False))
    new_version = changed.select(*project("n_", eff, high, True))

    brand_new = j.filter(F.col("o_h").isNull() & F.col("n_h").isNotNull()) \
                 .select(*project("n_", eff, high, True))

    # Policy: a key absent from the source stays open (no implicit delete).
    vanished = j.filter(F.col("n_h").isNull() & F.col("o_h").isNotNull()) \
                .select(*project("o_", F.col("o_valid_from"), high, True))

    return (history
            .unionByName(unchanged).unionByName(closed_old)
            .unionByName(new_version).unionByName(brand_new)
            .unionByName(vanished))
```

### Exercise 14

1. A brand-new key produces one open row with `valid_from = effective_date`.
2. An **unchanged** record stays one row and — critically — its **original `valid_from` is preserved**, not reset to today.
3. A **changed** record produces two rows: old closed at `effective_date` with `is_current = false`, new open.
4. **Idempotency:** applying the same incoming batch twice produces the identical table.
5. A key that **disappears** from the source stays open (per the stated policy).
6. **NULL vs empty string** are treated as different values — a change from `NULL` to `""` is a real change.
7. **Invariant:** at all times, exactly one `is_current = true` row per key.
8. A full three-day sequence produces a clean, gapless history.

<details>
<summary><b>Solution 14</b></summary>

```python
import pytest
from etl.scd import apply_scd2, HIGH_DATE

pytestmark = pytest.mark.spark

TRACKED = ["email", "country"]
DIM_SCHEMA = ("customer_id string, email string, country string, "
              "valid_from date, valid_to date, is_current boolean")
SRC_SCHEMA = "customer_id string, email string, country string"


@pytest.fixture
def empty_dim(spark):
    return spark.createDataFrame([], DIM_SCHEMA)


@pytest.fixture
def make_source(spark):
    def _make(rows):
        return spark.createDataFrame(rows, SRC_SCHEMA)
    return _make


def as_dicts(df):
    return sorted(
        (r.asDict() for r in df.collect()),
        key=lambda d: (d["customer_id"], str(d["valid_from"])),
    )


def assert_one_current_per_key(df, key="customer_id"):
    dupes = (df.filter(F_is_current(df)).groupBy(key).count()
               .filter("count > 1").count())
    assert dupes == 0, "invariant violated: multiple current rows for a key"


def F_is_current(df):
    return df["is_current"]


# ---------- tests ----------

def test_new_key_inserted(empty_dim, make_source):
    out = apply_scd2(empty_dim, make_source([("c1", "a@x.com", "US")]),
                     "customer_id", TRACKED, "2024-01-01")
    rows = as_dicts(out)
    assert len(rows) == 1
    assert rows[0]["is_current"] is True
    assert str(rows[0]["valid_from"]) == "2024-01-01"
    assert str(rows[0]["valid_to"]) == HIGH_DATE


def test_unchanged_preserves_original_valid_from(empty_dim, make_source):
    src = make_source([("c1", "a@x.com", "US")])
    day1 = apply_scd2(empty_dim, src, "customer_id", TRACKED, "2024-01-01")
    day2 = apply_scd2(day1, src, "customer_id", TRACKED, "2024-01-02")

    rows = as_dicts(day2)
    assert len(rows) == 1
    assert str(rows[0]["valid_from"]) == "2024-01-01"     # NOT 2024-01-02
    assert rows[0]["is_current"] is True


def test_change_closes_old_and_opens_new(empty_dim, make_source):
    day1 = apply_scd2(empty_dim, make_source([("c1", "old@x.com", "US")]),
                      "customer_id", TRACKED, "2024-01-01")
    day2 = apply_scd2(day1, make_source([("c1", "new@x.com", "US")]),
                      "customer_id", TRACKED, "2024-06-01")

    rows = as_dicts(day2)
    assert len(rows) == 2

    old, new = rows[0], rows[1]
    assert old["email"] == "old@x.com"
    assert old["is_current"] is False
    assert str(old["valid_from"]) == "2024-01-01"
    assert str(old["valid_to"]) == "2024-06-01"

    assert new["email"] == "new@x.com"
    assert new["is_current"] is True
    assert str(new["valid_from"]) == "2024-06-01"
    assert str(new["valid_to"]) == HIGH_DATE

    assert_one_current_per_key(day2)


def test_is_idempotent(empty_dim, make_source):
    src = make_source([("c1", "a@x.com", "US"), ("c2", "b@x.com", "CA")])
    once = apply_scd2(empty_dim, src, "customer_id", TRACKED, "2024-01-01")
    twice = apply_scd2(once, src, "customer_id", TRACKED, "2024-01-01")
    assert as_dicts(once) == as_dicts(twice)


def test_rerun_on_later_date_still_idempotent(empty_dim, make_source):
    """Re-running tomorrow with identical source must not create a version."""
    src = make_source([("c1", "a@x.com", "US")])
    d1 = apply_scd2(empty_dim, src, "customer_id", TRACKED, "2024-01-01")
    d2 = apply_scd2(d1, src, "customer_id", TRACKED, "2024-01-02")
    d3 = apply_scd2(d2, src, "customer_id", TRACKED, "2024-01-03")
    assert d3.count() == 1


def test_vanished_key_stays_open(empty_dim, make_source):
    d1 = apply_scd2(empty_dim, make_source([("c1", "a@x.com", "US"),
                                            ("c2", "b@x.com", "CA")]),
                    "customer_id", TRACKED, "2024-01-01")
    d2 = apply_scd2(d1, make_source([("c1", "a@x.com", "US")]),
                    "customer_id", TRACKED, "2024-02-01")

    c2 = [r for r in as_dicts(d2) if r["customer_id"] == "c2"]
    assert len(c2) == 1
    assert c2[0]["is_current"] is True
    assert str(c2[0]["valid_from"]) == "2024-01-01"


def test_null_and_empty_string_are_different(empty_dim, make_source):
    d1 = apply_scd2(empty_dim, make_source([("c1", None, "US")]),
                    "customer_id", TRACKED, "2024-01-01")
    d2 = apply_scd2(d1, make_source([("c1", "", "US")]),
                    "customer_id", TRACKED, "2024-02-01")
    assert d2.count() == 2, "NULL -> '' must be recorded as a change"


def test_null_to_null_is_not_a_change(empty_dim, make_source):
    src = make_source([("c1", None, "US")])
    d1 = apply_scd2(empty_dim, src, "customer_id", TRACKED, "2024-01-01")
    d2 = apply_scd2(d1, src, "customer_id", TRACKED, "2024-02-01")
    assert d2.count() == 1


def test_three_day_history_is_gapless(empty_dim, make_source):
    d1 = apply_scd2(empty_dim, make_source([("c1", "v1@x.com", "US")]),
                    "customer_id", TRACKED, "2024-01-01")
    d2 = apply_scd2(d1, make_source([("c1", "v2@x.com", "US")]),
                    "customer_id", TRACKED, "2024-02-01")
    d3 = apply_scd2(d2, make_source([("c1", "v3@x.com", "US")]),
                    "customer_id", TRACKED, "2024-03-01")

    rows = as_dicts(d3)
    assert len(rows) == 3
    windows = [(str(r["valid_from"]), str(r["valid_to"])) for r in rows]
    assert windows == [("2024-01-01", "2024-02-01"),
                       ("2024-02-01", "2024-03-01"),
                       ("2024-03-01", HIGH_DATE)]
    assert sum(r["is_current"] for r in rows) == 1
```

**`test_unchanged_preserves_original_valid_from` is the test that catches the most common SCD2 bug.** A naive implementation rewrites `valid_from` on every run. Nothing errors. Row counts look right. Then someone asks "how many customers were in the EU segment on March 1st?" and every answer is wrong, because every record now claims to have started yesterday.

**`test_null_and_empty_string_are_different` explains the `<NULL>` sentinel** in `_row_hash`. `concat_ws` *skips* nulls, so without the `coalesce`, `(NULL, "US")` and `("", "US")` both hash the string `"US"` — and a real attribute change is invisible forever. This is a genuinely nasty silent bug.

**`test_rerun_on_later_date_still_idempotent`** covers what actually happens in production: your DAG runs daily, the dimension changes rarely. If a same-source rerun creates a version, you get 365 identical rows per customer per year.

**`assert_one_current_per_key` is an invariant, not a case.** Call it at the end of every SCD2 test you write. Two current rows for one key means every downstream join fans out and every metric doubles.

**Design decisions this code makes that yours might not:** `valid_to` of the closed row equals `valid_from` of the new one (touching windows, not gapped — pick one and be consistent, or your `BETWEEN` queries double-count on the boundary day). Vanished keys stay open rather than being soft-deleted. Write a test for whichever policy *you* chose; the point is that the choice is visible.
</details>

---

## Level 15 — Timezones and partition dates

### Concept

The single most common "the numbers are slightly off" bug in analytics: a UTC timestamp assigned to the wrong local calendar day. Every event between midnight and 5am UTC belongs to *yesterday* in New York. Get this wrong and every daily report is off by a few hours' worth of data — small enough that nobody notices for a year.

### Code under test

```python
# etl/spark_jobs.py
def add_partition_date(df: DataFrame, ts_col: str = "event_ts",
                       tz: str = "America/New_York") -> DataFrame:
    """Assign each UTC event to its local business day."""
    return df.withColumn(
        "partition_date",
        F.to_date(F.from_utc_timestamp(F.col(ts_col), tz)),
    )
```

### Exercise 15

1. A midday UTC timestamp maps to the same calendar day.
2. `2024-06-01T03:00:00Z` maps to **`2024-05-31`** in New York (the off-by-one-day case).
3. `2024-06-01T23:00:00Z` maps to `2024-06-01` in New York but `2024-06-02` in Tokyo.
4. **Fall-back DST:** `2024-11-03T05:30:00Z` and `2024-11-03T06:30:00Z` are both `01:30` local — both land on `2024-11-03`.
5. **Spring-forward DST:** `2024-03-10T07:30:00Z` lands on `2024-03-10`.
6. A **null** timestamp gives a null partition — not `1970-01-01`.
7. The result does **not** depend on the machine's local timezone.

<details>
<summary><b>Solution 15</b></summary>

```python
from datetime import datetime
import pytest
from etl.spark_jobs import add_partition_date

pytestmark = pytest.mark.spark


@pytest.fixture
def make_events(spark):
    def _make(iso_timestamps):
        rows = [(f"e{i}", datetime.fromisoformat(t) if t else None)
                for i, t in enumerate(iso_timestamps)]
        return spark.createDataFrame(rows, "event_id string, event_ts timestamp")
    return _make


def partitions(df):
    return [str(r.partition_date) if r.partition_date else None
            for r in df.orderBy("event_id").collect()]


@pytest.mark.parametrize("utc_ts,tz,expected", [
    ("2024-06-01T12:00:00", "America/New_York", "2024-06-01"),
    ("2024-06-01T03:00:00", "America/New_York", "2024-05-31"),  # off-by-one day
    ("2024-06-01T04:00:00", "America/New_York", "2024-06-01"),  # exact boundary
    ("2024-06-01T23:00:00", "America/New_York", "2024-06-01"),
    ("2024-06-01T23:00:00", "Asia/Tokyo",       "2024-06-02"),  # next day in JST
    ("2024-06-01T12:00:00", "UTC",              "2024-06-01"),
    ("2024-06-01T00:00:00", "Asia/Kolkata",     "2024-06-01"),  # +05:30 offset
])
def test_partition_assignment(make_events, utc_ts, tz, expected):
    df = add_partition_date(make_events([utc_ts]), tz=tz)
    assert partitions(df) == [expected]


def test_dst_fall_back_ambiguous_hour(make_events):
    """2024-11-03 01:30 EDT and 01:30 EST are different UTC instants,
    the same local wall-clock time, and the same business day."""
    df = add_partition_date(
        make_events(["2024-11-03T05:30:00", "2024-11-03T06:30:00"]),
        tz="America/New_York",
    )
    assert partitions(df) == ["2024-11-03", "2024-11-03"]


def test_dst_spring_forward(make_events):
    """2024-03-10 02:00-03:00 local does not exist; nothing should break."""
    df = add_partition_date(
        make_events(["2024-03-10T06:30:00", "2024-03-10T07:30:00"]),
        tz="America/New_York",
    )
    assert partitions(df) == ["2024-03-10", "2024-03-10"]


def test_dst_shifts_the_midnight_boundary(make_events):
    """04:00 UTC is midnight in winter (EST) but 23:00 the previous day
    is midnight in summer (EDT). The boundary MOVES."""
    winter = add_partition_date(make_events(["2024-01-15T04:30:00"]),
                                tz="America/New_York")
    summer = add_partition_date(make_events(["2024-07-15T03:30:00"]),
                                tz="America/New_York")
    assert partitions(winter) == ["2024-01-14"]   # 23:30 EST on the 14th
    assert partitions(summer) == ["2024-07-14"]   # 23:30 EDT on the 14th


def test_null_timestamp_gives_null_partition(make_events):
    df = add_partition_date(make_events([None]))
    assert partitions(df) == [None]


def test_result_independent_of_session_timezone(spark, make_events):
    original = spark.conf.get("spark.sql.session.timeZone")
    try:
        results = []
        for session_tz in ["UTC", "Asia/Kolkata", "America/Los_Angeles"]:
            spark.conf.set("spark.sql.session.timeZone", session_tz)
            df = add_partition_date(make_events(["2024-06-01T03:00:00"]),
                                    tz="America/New_York")
            results.append(partitions(df)[0])
        assert len(set(results)) == 1, f"result varied by session tz: {results}"
    finally:
        spark.conf.set("spark.sql.session.timeZone", original)
```

**`test_result_independent_of_session_timezone` is a portability test.** It catches the code that works on your laptop and produces different partitions in the cluster. Note the `try/finally` — if you mutate global Spark config in a session-scoped fixture's session, you *must* restore it, or you corrupt every test that runs after. (A cleaner alternative: a fixture that sets and restores the config with `yield`.)

**`test_dst_shifts_the_midnight_boundary` is the subtle one.** People often "fix" timezone bugs by subtracting a constant 5 hours. That's correct for half the year and wrong for the other half. Only a real timezone database handles DST, and only a test proves you're using one.

**Rule to adopt:** store timestamps in UTC everywhere, convert to local only at the point where a business day is assigned, and pass the timezone in as an explicit parameter rather than reading the server's locale. Then test all three: the conversion, the boundary, and the DST transitions.
</details>

---

## Level 16 — A data quality rule suite

### Concept

Once you have transforms tested, you want reusable *data* assertions — not-null, uniqueness, ranges, referential integrity, freshness. Parametrizing over a list of rule objects gives you one test that reports each rule separately.

The trap this level teaches: **three-valued logic**. In SQL and Spark, `NOT (NULL > 0)` is `NULL`, not `TRUE`. A naive "count the rows that fail" filter silently skips every null row — so your null-detection check misses nulls.

### Code under test

```python
# etl/quality.py
from dataclasses import dataclass
from typing import Callable
from pyspark.sql import DataFrame, Column, functions as F


@dataclass(frozen=True)
class Check:
    name: str
    condition: Callable[[], Column]     # True == row is VALID
    severity: str = "error"


def count_violations(df: DataFrame, check: Check) -> int:
    """Null-safe: a condition evaluating to NULL counts as a violation."""
    valid = F.coalesce(check.condition(), F.lit(False))
    return df.filter(~valid).count()


def run_checks(df: DataFrame, checks) -> dict:
    return {c.name: count_violations(df, c) for c in checks}


ORDER_CHECKS = [
    Check("order_id_not_null",   lambda: F.col("order_id").isNotNull()),
    Check("amount_positive",     lambda: F.col("amount") > 0),
    Check("qty_reasonable",      lambda: F.col("qty").between(1, 1000)),
    Check("status_in_enum",      lambda: F.col("status").isin("new", "paid", "shipped")),
    Check("currency_iso",        lambda: F.col("currency").rlike("^[A-Z]{3}$")),
    Check("email_has_at",        lambda: F.col("email").contains("@"), "warn"),
]
```

### Exercise 16

1. A fully clean batch produces zero violations for every rule.
2. Parametrize over `ORDER_CHECKS` so each rule is a separate test result, each fed one deliberately bad row.
3. **The null trap:** a row with `amount = NULL` must be counted as violating `amount_positive`.
4. Uniqueness is a *dataset-level* check, not a row-level one — write it separately.
5. Referential integrity: order rows whose `customer_id` isn't in the dimension.
6. A `severity="warn"` rule failing does not fail the pipeline, but an `"error"` one does.

<details>
<summary><b>Solution 16</b></summary>

```python
import pytest
from pyspark.sql import functions as F
from etl.quality import Check, count_violations, run_checks, ORDER_CHECKS

pytestmark = pytest.mark.spark

SCHEMA = ("order_id string, customer_id string, amount double, qty int, "
          "status string, currency string, email string")

CLEAN = ("o1", "c1", 100.0, 2, "paid", "USD", "a@x.com")


@pytest.fixture
def make_batch(spark):
    def _make(*rows):
        return spark.createDataFrame(list(rows) or [CLEAN], SCHEMA)
    return _make


def bad(**overrides):
    fields = ["order_id", "customer_id", "amount", "qty",
              "status", "currency", "email"]
    row = dict(zip(fields, CLEAN))
    row.update(overrides)
    return tuple(row[f] for f in fields)


def test_clean_batch_has_no_violations(make_batch):
    assert run_checks(make_batch(CLEAN), ORDER_CHECKS) == {
        c.name: 0 for c in ORDER_CHECKS
    }


@pytest.mark.parametrize("check_name,bad_row", [
    ("order_id_not_null", bad(order_id=None)),
    ("amount_positive",   bad(amount=-1.0)),
    ("amount_positive",   bad(amount=0.0)),
    ("qty_reasonable",    bad(qty=0)),
    ("qty_reasonable",    bad(qty=5000)),
    ("status_in_enum",    bad(status="refunded")),
    ("status_in_enum",    bad(status="PAID")),      # case matters
    ("currency_iso",      bad(currency="usd")),
    ("currency_iso",      bad(currency="US")),
    ("email_has_at",      bad(email="not-an-email")),
])
def test_each_rule_catches_its_bad_row(make_batch, check_name, bad_row):
    check = next(c for c in ORDER_CHECKS if c.name == check_name)
    assert count_violations(make_batch(bad_row), check) == 1


@pytest.mark.parametrize("null_field,check_name", [
    ("amount",   "amount_positive"),
    ("qty",      "qty_reasonable"),
    ("status",   "status_in_enum"),
    ("currency", "currency_iso"),
])
def test_nulls_count_as_violations(make_batch, null_field, check_name):
    """Three-valued logic trap: NOT(NULL > 0) is NULL, not TRUE."""
    check = next(c for c in ORDER_CHECKS if c.name == check_name)
    assert count_violations(make_batch(bad(**{null_field: None})), check) == 1


def test_violation_counts_are_accurate(make_batch):
    batch = make_batch(CLEAN, bad(amount=-1.0), bad(amount=None), bad(qty=0))
    results = run_checks(batch, ORDER_CHECKS)
    assert results["amount_positive"] == 2
    assert results["qty_reasonable"] == 1
    assert results["order_id_not_null"] == 0


# --- dataset-level checks ---

def test_primary_key_uniqueness(make_batch):
    batch = make_batch(bad(order_id="o1"), bad(order_id="o1"), bad(order_id="o2"))
    dupes = batch.groupBy("order_id").count().filter("count > 1").count()
    assert dupes == 1


def test_referential_integrity(spark, make_batch):
    orders = make_batch(bad(customer_id="c1"), bad(customer_id="ghost"))
    dim = spark.createDataFrame([("c1",)], "customer_id string")
    orphans = orders.join(dim, "customer_id", "left_anti")
    assert orphans.count() == 1
    assert orphans.collect()[0].customer_id == "ghost"


def test_row_count_within_expected_range(make_batch):
    """Volume anomaly detection: yesterday 100 rows, today 3 = upstream broke."""
    batch = make_batch(*[bad(order_id=f"o{i}") for i in range(50)])
    assert 10 <= batch.count() <= 1000


# --- severity handling ---

def test_warn_severity_does_not_block(make_batch):
    results = run_checks(make_batch(bad(email="nope")), ORDER_CHECKS)
    errors = [c.name for c in ORDER_CHECKS
              if c.severity == "error" and results[c.name] > 0]
    assert errors == []              # nothing blocking
    assert results["email_has_at"] == 1   # but it is reported


def test_error_severity_blocks(make_batch):
    results = run_checks(make_batch(bad(amount=-5.0)), ORDER_CHECKS)
    errors = [c.name for c in ORDER_CHECKS
              if c.severity == "error" and results[c.name] > 0]
    assert errors == ["amount_positive"]
```

**`test_nulls_count_as_violations` is the whole point of this level.** Without the `coalesce`, `df.filter(~(F.col("amount") > 0))` evaluates to `NOT NULL` = `NULL` for null amounts, the row is filtered out, and your check reports **zero violations on a column full of nulls**. Your data quality framework confidently tells you the data is fine. This exact bug ships in a lot of home-grown validation code.

**Note `("status_in_enum", bad(status="PAID"))`.** Case sensitivity in enum checks is a real production issue when an upstream system changes serialization. The test documents that `"PAID"` is *not* accepted — which is either correct or a bug you now get to decide about deliberately.

**Note `bad(amount=0.0)`.** Zero is the boundary of "positive". Refunds, voided orders, and $0 promotional line items all produce zeros, and whether they're valid is a business question. Testing zero forces the question.

**Row-level vs dataset-level:** uniqueness, referential integrity, row counts, and freshness cannot be expressed as a per-row boolean. Keep them as separate functions. Trying to force everything into one `Check` abstraction is a common over-engineering trap.

**When to graduate to a real tool:** this pattern is great for unit-testing your rules. For *production* data quality — running against full tables, storing results over time, alerting on drift — use dbt tests, Great Expectations, or Soda. pytest tests your code; those monitor your data. Both, not either.
</details>

---

## Level 17 — Capstone: the whole pipeline

### Concept

Unit tests prove each piece works. An **integration test** proves the pieces work *together*: real file in, real database out, no mocks in the middle. You want a handful of these, not hundreds — they're slower and they fail for many reasons at once.

### Code under test

```python
# etl/pipeline.py
import logging
from etl.ingest import read_orders_csv
from etl.load import init_schema

log = logging.getLogger(__name__)


def run_pipeline(csv_path, conn, batch_id):
    """Read -> validate -> split good/bad -> load -> record run metadata."""
    raw = read_orders_csv(csv_path)

    raw["_valid"] = (
        raw["order_id"].notna()
        & raw["customer_id"].notna()
        & raw["amount"].apply(lambda v: _is_positive_number(v))
    )
    good = raw[raw["_valid"]].drop(columns=["_valid"])
    bad = raw[~raw["_valid"]].drop(columns=["_valid"])

    good = good.drop_duplicates(subset=["order_id"], keep="last")

    conn.execute("BEGIN TRANSACTION")
    try:
        conn.execute("DELETE FROM orders WHERE batch_id = ?", [batch_id])
        for _, r in good.iterrows():
            conn.execute(
                "INSERT INTO orders VALUES (?, ?, ?, ?)",
                [r["order_id"], r["customer_id"], float(r["amount"]), batch_id],
            )
        for _, r in bad.iterrows():
            conn.execute(
                "INSERT INTO orders_rejected VALUES (?, ?, ?)",
                [str(r.get("order_id")), str(r.to_dict()), batch_id],
            )
        conn.execute("COMMIT")
    except Exception:
        conn.execute("ROLLBACK")
        log.exception("Batch %s failed and was rolled back", batch_id)
        raise

    return {"loaded": len(good), "rejected": len(bad), "batch_id": batch_id}


def _is_positive_number(v):
    try:
        return float(v) > 0
    except (TypeError, ValueError):
        return False
```

### Exercise 17

Write an integration test suite that:

1. Loads a clean CSV end to end and asserts the rows land in the warehouse.
2. **Reruns the identical file** and asserts the table is unchanged (the `DELETE ... WHERE batch_id` makes the batch idempotent).
3. Splits bad rows into `orders_rejected` while still loading the good ones.
4. Handles a header-only file: zero loaded, zero rejected, no crash.
5. Rolls back completely when the load raises partway through.
6. Handles a file with duplicate `order_id`s.
7. Confirms `loaded + rejected` equals the input row count — **nothing silently vanishes**.

<details>
<summary><b>Solution 17</b></summary>

```python
import duckdb
import pytest
from etl.pipeline import run_pipeline

pytestmark = pytest.mark.integration

DDL = """
CREATE TABLE orders (
    order_id VARCHAR, customer_id VARCHAR, amount DOUBLE, batch_id VARCHAR
);
CREATE TABLE orders_rejected (
    order_id VARCHAR, raw_row VARCHAR, batch_id VARCHAR
);
"""

HEADER = "order_id,customer_id,amount\n"


@pytest.fixture
def conn():
    c = duckdb.connect(":memory:")
    c.execute(DDL)
    yield c
    c.close()


@pytest.fixture
def csv(tmp_path):
    def _write(body, name="orders.csv"):
        p = tmp_path / name
        p.write_text(HEADER + body, encoding="utf-8")
        return p
    return _write


def orders(conn):
    return conn.execute(
        "SELECT order_id, customer_id, amount, batch_id FROM orders ORDER BY order_id"
    ).fetchall()


def rejected(conn):
    return conn.execute("SELECT order_id FROM orders_rejected").fetchall()


def test_happy_path(conn, csv):
    path = csv("o1,c1,100.0\no2,c2,250.5\n")
    result = run_pipeline(path, conn, "b1")

    assert result == {"loaded": 2, "rejected": 0, "batch_id": "b1"}
    rows = orders(conn)
    assert len(rows) == 2
    assert rows[0] == ("o1", "c1", 100.0, "b1")


def test_rerun_is_idempotent(conn, csv):
    path = csv("o1,c1,100.0\no2,c2,250.5\n")

    run_pipeline(path, conn, "b1")
    first = orders(conn)

    run_pipeline(path, conn, "b1")       # exact same batch id, same file
    second = orders(conn)

    assert first == second
    assert len(second) == 2              # not 4


def test_different_batch_ids_accumulate(conn, csv):
    run_pipeline(csv("o1,c1,10.0\n", "a.csv"), conn, "b1")
    run_pipeline(csv("o2,c2,20.0\n", "b.csv"), conn, "b2")
    assert len(orders(conn)) == 2


def test_bad_rows_are_quarantined_not_dropped(conn, csv):
    path = csv(
        "o1,c1,100.0\n"        # good
        "o2,c2,-5.0\n"         # negative amount
        "o3,c3,abc\n"          # unparseable
        ",c4,50.0\n"           # missing order_id
        "o5,,50.0\n"           # missing customer_id
    )
    result = run_pipeline(path, conn, "b1")

    assert result["loaded"] == 1
    assert result["rejected"] == 4
    assert len(orders(conn)) == 1
    assert len(rejected(conn)) == 4


def test_nothing_silently_vanishes(conn, csv):
    """The accounting identity: in == loaded + rejected."""
    body = "o1,c1,100.0\no2,c2,-5.0\no3,c3,abc\no4,c4,75.0\n"
    result = run_pipeline(csv(body), conn, "b1")
    assert result["loaded"] + result["rejected"] == 4


def test_header_only_file(conn, csv):
    result = run_pipeline(csv(""), conn, "b1")
    assert result == {"loaded": 0, "rejected": 0, "batch_id": "b1"}
    assert orders(conn) == []


def test_duplicate_order_ids_deduplicated(conn, csv):
    path = csv("o1,c1,100.0\no1,c1,999.0\no2,c2,50.0\n")
    result = run_pipeline(path, conn, "b1")
    assert result["loaded"] == 2
    amounts = {r[0]: r[2] for r in orders(conn)}
    assert amounts["o1"] == 999.0        # keep="last"


def test_failure_rolls_back_entirely(conn, csv, monkeypatch):
    run_pipeline(csv("o1,c1,10.0\n", "first.csv"), conn, "b0")
    before = orders(conn)

    original_execute = conn.execute
    calls = {"n": 0}

    def flaky(sql, *args, **kwargs):
        if sql.strip().upper().startswith("INSERT INTO ORDERS "):
            calls["n"] += 1
            if calls["n"] == 2:
                raise RuntimeError("simulated warehouse failure")
        return original_execute(sql, *args, **kwargs)

    monkeypatch.setattr(conn, "execute", flaky)

    with pytest.raises(RuntimeError, match="simulated warehouse failure"):
        run_pipeline(csv("o2,c2,1.0\no3,c3,2.0\no4,c4,3.0\n", "second.csv"),
                     conn, "b1")

    monkeypatch.undo()
    assert orders(conn) == before        # no partial batch left behind


def test_failure_is_logged(conn, csv, caplog, monkeypatch):
    import logging
    original_execute = conn.execute

    def boom(sql, *args, **kwargs):
        if sql.strip().upper().startswith("INSERT INTO ORDERS "):
            raise RuntimeError("boom")
        return original_execute(sql, *args, **kwargs)

    monkeypatch.setattr(conn, "execute", boom)
    with caplog.at_level(logging.ERROR):
        with pytest.raises(RuntimeError):
            run_pipeline(csv("o1,c1,10.0\n"), conn, "b9")

    assert any("b9" in r.getMessage() for r in caplog.records)
```

**`test_rerun_is_idempotent` and `test_failure_rolls_back_entirely` are the two tests that let you sleep.** Together they mean: any failed run can be safely retried, and a failed run never leaves the warehouse in a half-state. That's the whole operational contract of a batch pipeline.

**`test_nothing_silently_vanishes` is an accounting identity, and it's underrated.** Row counts in must equal rows loaded plus rows rejected. Add this assertion to every pipeline you build. A row that is neither loaded nor rejected has been silently eaten by a filter, and you will only find out when someone reconciles against the source months later.

**`test_bad_rows_are_quarantined_not_dropped`** encodes the design choice to quarantine rather than crash. The alternative — fail the whole batch on one bad row — is also valid (and better for financial data). Either way, the test makes the policy explicit and reviewable.

**Keep integration tests few.** Five to ten per pipeline. They're slower and when they fail they tell you "something broke" rather than "this function broke." Mark them `@pytest.mark.integration` so `pytest -m "not integration"` gives you a fast inner loop.
</details>

---

# Appendix

## A. The bugs these tests exist to catch

A checklist to run against any transform you write:

**Input shape**
- Empty input (zero rows) — does it crash or return an empty frame with the right schema?
- Single row — do your window functions and aggregations still work?
- All-null column — does type inference fall over?
- Only-header file, completely empty file

**Keys and identity**
- Null primary key
- Duplicate primary key
- Key needing trim/case normalization to match
- Key that exists on the fact side but not the dimension (and vice versa)
- Composite keys where one part is null

**Numbers**
- Zero (the falsy trap)
- Negative
- Exactly at every threshold, and one unit either side
- Very large — does `int32` overflow, does `decimal(10,2)` fit?
- Float rounding at the half-cent
- Division by zero / by a null

**Strings**
- Leading and trailing whitespace
- Case differences
- Empty string vs NULL (never the same thing)
- Non-ASCII, emoji, BOM
- Embedded delimiters and newlines in CSV
- Leading zeros that must survive

**Time**
- UTC vs local business day
- DST spring-forward and fall-back
- Timestamp vs date precision
- Events arriving out of order
- Events arriving late, after a partition was already written
- Year boundaries, month-end, leap day

**Pipeline behaviour**
- Rerunning the same batch (idempotency)
- Failure midway (atomicity)
- Upstream schema drift: new column, renamed column, retyped column
- Zero-row incremental batch
- Volume anomaly — 10x the usual rows, or 1/10th

## B. What *not* to test

- **That Spark/pandas/your database works.** Don't test `df.filter()`. Test your business rule.
- **Framework glue.** Don't unit-test that your Airflow DAG has tasks in it.
- **Getters and constructors** with no logic.
- **Production data.** Tests must be deterministic; production data isn't. Use tiny hand-built fixtures.
- **Every permutation.** Test boundaries and representative cases, not all 10,000 combinations.

Coverage percentage is a weak signal. Ten tests on your SCD2 logic beat two hundred on your data classes.

## C. Built-in fixtures worth knowing

| Fixture | Gives you |
|---|---|
| `tmp_path` | Fresh temp directory as a `Path` |
| `tmp_path_factory` | Session-scoped version |
| `monkeypatch` | Auto-reverting env/attr patching |
| `capsys` / `capfd` | Captured stdout/stderr |
| `caplog` | Captured log records |
| `recwarn` | Captured warnings |
| `request` | Metadata about the running test |

## D. Fixture scopes

| Scope | Created once per | Use for |
|---|---|---|
| `function` (default) | test | DataFrames, sample data, anything mutable |
| `class` | test class | shared setup within a group |
| `module` | file | a moderately expensive read-only resource |
| `session` | whole run | SparkSession, docker container, warehouse connection |

**Rule of thumb:** expensive and read-only → `session`. Cheap or mutable → `function`. A session-scoped mutable fixture is a "passes alone, fails in the suite" bug waiting to happen.

## E. Markers and CI

```python
@pytest.mark.slow
@pytest.mark.spark
@pytest.mark.integration
@pytest.mark.skip(reason="not implemented yet")
@pytest.mark.skipif(sys.version_info < (3, 10), reason="needs 3.10+")
@pytest.mark.xfail(reason="known bug, ticket DATA-142")
```

A sensible CI split:

```yaml
- run: pytest -m "not spark and not integration"   # seconds, runs on every push
- run: pytest -m "spark"                           # minutes, runs on PR
- run: pytest -m "integration"                     # runs pre-merge only
```

`xfail` is genuinely useful in data work: when you find a bug, write the failing test, mark it `xfail` with the ticket number, and merge. The moment someone fixes it, pytest reports `XPASS` and you know to remove the marker.

## F. Common beginner mistakes

| Symptom | Cause |
|---|---|
| "collected 0 items" | File or function not named `test_*` |
| `ModuleNotFoundError: etl` | Missing `__init__.py`, or not installed with `pip install -e .` |
| Test passes alone, fails in suite | Shared mutable state — a module/session-scoped fixture being mutated |
| Test fails only in CI | Environment variable, timezone, or locale difference — clear them explicitly |
| Spark suite takes 10 minutes | `shuffle.partitions` still at 200, or a function-scoped SparkSession |
| Flaky row-order failures | Asserting order without `orderBy`; use `ignore_row_order=True` |
| Nullability mismatches in chispa | Add `ignore_nullable=True` |
| Float comparison fails by 1e-17 | Use `pytest.approx` or `Decimal` |
| `pytest.raises` passes for the wrong reason | Exception class too broad, or two statements inside the `with` |

## G. A four-week practice plan

**Week 1 — foundations.** Levels 1–6. Then go to your own codebase, find the smallest pure function, and write five tests for it including the empty and boundary cases.

**Week 2 — the outside world.** Levels 7–10. Pick one real file reader and one real load function from your project and test them properly. Write the idempotency test.

**Week 3 — Spark.** Levels 11–13. Set up the `conftest.py` session fixture in your repo. Add a schema contract test for your most important source table.

**Week 4 — the hard stuff.** Levels 14–17. Take your most complex existing transform and write tests for it. You will find at least one bug. That's the point.

**Then, permanently:** when a pipeline breaks in production, write the failing test *first*, watch it fail, then fix the code. That single habit is what turns a test suite from a chore into the thing that stops you being paged twice for the same reason.
