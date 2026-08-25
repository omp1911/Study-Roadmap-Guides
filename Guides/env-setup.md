# Setup

One Docker container for all four guides. Python, JavaScript, TypeScript, Spark, a browser IDE and notebooks — nothing on your laptop except Docker.

**Time:** ten minutes, most of it waiting for downloads.  
**You need:** Docker Desktop (macOS/Windows) or Docker Engine + Compose plugin (Linux).

---

## How to use this guide

Do this once. Every other guide assumes it's finished and starts at concept 1.

| Where you work | Address | Good for |
|---|---|---|
| **IDE** | `http://localhost:8443` | Everything. Real VS Code — editor, file tree, terminal, notebooks, debugger. |
| **Notebooks** | `http://localhost:8888` | JupyterLab, if you prefer it to the IDE's notebook view. |
| **Shell** | `docker compose exec lab bash` | When you'd rather type than click. |
| **dbt docs** | `http://localhost:8080` | Guide 6's lineage graph, once you run `dbt docs serve`. |

Three doors, one room. Same files, same Python, same Node.

**Two rules that make the rest of this painless:**

1. **To add a Python library**, add a line to `requirements.txt`. To add a JS one, add it to `package.json`. Then `docker compose restart lab`. Never edit `docker-compose.yml` for this.
2. **Your files are never at risk.** They live in your folder on your laptop; the container just looks at them. Restart, rebuild, `down -v`, delete the image — notebooks, databases and code are untouched.

**Guide 4 (Foundry)** runs in your own Foundry workspace, not here.

---

## Contents

| # | Step | What you'll have after it |
|---|---|---|
| 1 | [Get the files in one folder](#1-get-the-files-in-one-folder) | A working folder Docker can see |
| 2 | [Start the container](#2-start-the-container) | Everything installed, IDE and notebooks live |
| 3 | [Adding libraries later](#3-adding-libraries-later) | The one workflow you'll use constantly |
| 4 | [The IDE](#4-the-ide) | A Codespaces-like editor, already configured |
| 5 | [JavaScript and TypeScript](#5-javascript-and-typescript) | A real TS project with watch mode and type-checking |
| 6 | [Seed the toy shop database](#6-seed-the-toy-shop-database) | `shop.duckdb` for guide 1 |
| 7 | [Create the etl-practice project](#7-create-the-etl-practice-project) | The layout guide 3 tests against |
| 8 | [The warehouse and dbt](#8-the-warehouse-and-dbt) | Landing files for guides 5 and 6 |
| 9 | [Daily commands](#9-daily-commands) | The handful you'll actually use |
| 10 | [Troubleshooting](#10-troubleshooting) | Fixes for the things that go wrong |

---

## 1. Get the files in one folder

Everything in one folder. That folder becomes `/work` inside the container.

```
de-practice/
├── docker-compose.yml       ← you rarely touch this
├── requirements.txt         ← Python libraries. edit freely.
├── package.json             ← JS/TS libraries. edit freely.
├── 00-setup.md
├── 01-data-modelling.md
├── 02-python-async-regex-patterns.md
├── 03-pytest-for-etl.md
├── 04-foundry-osdk-marketplace.md
├── 05-medallion-architecture.md
└── 06-dbt-basics.md
```

```bash
cd de-practice
docker --version          # any 24.x or newer
docker compose version    # must say "Docker Compose version v2..."
```

> **Trap.** If `docker compose version` fails but `docker-compose --version` works, you have the old standalone v1. Everything below still works — write `docker-compose` in place of `docker compose`.

---

## 2. Start the container

```bash
docker compose up -d
docker compose logs -f lab
```

First run: **five to eight minutes.** It downloads Python, then installs Java, Node, everything in `requirements.txt`, everything in `package.json`, the TypeScript toolchain, the IDE and its extensions. All of that lands in Docker volumes, so later starts take about ten seconds.

Watch for this, then `Ctrl-C` to stop tailing:

```
[lab] READY
        IDE        http://localhost:8443
        Notebooks  http://localhost:8888
        Shell      docker compose exec lab bash
```

Open `http://localhost:8443`. That's your workspace.

### Check it

In the IDE's terminal (*Terminal → New Terminal*):

```bash
python -c "import duckdb, pandas, pyspark; print('python ok', duckdb.__version__)"
pytest --version
java -version                        # PySpark won't start without this
node --version && tsc --version      # JS and TS
```

> **Why no password on the IDE?** The ports publish as `127.0.0.1:8443`, not `8443` — reachable from your own browser and from nowhere else, not from your phone, not from café wifi. If you ever want access from another machine, add a password *before* widening that binding.

---

## 3. Adding libraries later

This is the workflow you'll use most, so it's worth doing once deliberately.

```
   You edit                      You run                        What happens
 ┌──────────────────┐      ┌──────────────────────┐      ┌──────────────────────┐
 │ requirements.txt │      │ docker compose       │      │ new packages install │
 │      or          │ ───▶ │      restart lab     │ ───▶ │ ~20 seconds          │
 │ package.json     │      └──────────────────────┘      │                      │
 └──────────────────┘                                    │ your files, notebooks│
                                                         │ and databases: as    │
                                                         │ you left them        │
                                                         └──────────────────────┘
```

**Python.** Open `requirements.txt`, add the name under *your libraries*, save:

```
# --- your libraries ---
polars
sqlalchemy
```

```bash
docker compose restart lab
```

Wait for `READY` in `docker compose logs -f lab`, then reload the browser tab. `import polars` works.

**JavaScript / TypeScript.** Add to `package.json` under `dependencies`, then the same restart. Or, from the IDE terminal, the shortcut that does both at once:

```bash
npm install zod          # installs now AND writes it into package.json
```

That second form is worth preferring for JS — npm updates the file for you, so the next rebuild already knows about it.

**Pin a version when you care, don't when you don't.** `pandas==2.2.2` in `requirements.txt` means everyone who runs this gets that exact version; a bare `pandas` means you get whatever's current. For practice, bare is fine. For anything you'd hand to a colleague, pin.

> **Tip.** `pip install polars` in the terminal also works and is instant — but it's gone the next time you rebuild. Use it to try something; put it in `requirements.txt` once you've decided to keep it.

---

## 4. The IDE

`http://localhost:8443` is code-server: VS Code, running in the container, in your browser. It comes configured — Python interpreter already pointed at `/opt/venv/bin/python`, pytest already enabled, format-on-save already on. Extensions for Python, Jupyter, ESLint, Prettier, SQLTools and YAML install on first run.

What you get that a terminal doesn't give you:

| | |
|---|---|
| **Run a test by clicking it** | The flask icon in the sidebar lists every test. Click the ▶ next to one to run it, the 🐞 to debug it. Guide 3 is much easier this way. |
| **Notebooks inside the editor** | Make a `.ipynb` file and it opens as a notebook, with the file tree and your source next to it. |
| **A real debugger** | Click left of a line number for a breakpoint, then F5. Inspect variables instead of adding `print` statements. |
| **Type errors as you type** | Red squiggles in `.ts` files, no build step needed. |
| **Integrated terminal** | ``Ctrl-` `` — same shell as `docker compose exec lab bash`. |

To change any setting, edit `.vscode/settings.json` in your folder. It's yours; the container creates it once and never touches it again.

> **Tip.** Your IDE settings, extensions and open-file history live in a Docker volume, so plain `docker compose down` keeps them. Only `down -v` resets the editor.

---

## 5. JavaScript and TypeScript

First start creates a real TypeScript project — not a scratch folder:

```
tsconfig.json          strict mode, ES2022, node types
ts/index.ts            your entry point
package.json           dependencies and the scripts below
node_modules/          installed packages
```

From the IDE terminal:

| Command | What it does |
|---|---|
| `npm run dev` | Runs `ts/index.ts` and **re-runs it every time you save**. This is the practice loop. |
| `npm run check` | Type-check the whole project without running it. |
| `npm run test` | Vitest. Put tests in `ts/*.test.ts`. |
| `npm run serve` | Vite dev server on `http://localhost:5173`, already bound correctly. |
| `tsx some-file.ts` | Run any single `.ts` file immediately, no config. |

Try it — open `ts/index.ts` in the IDE, run `npm run dev` in the terminal, then change a line and save. Output updates without you touching the terminal.

> **Trap.** `tsx` and `npm run dev` **strip types and run**; they do not type-check. A file with genuine type errors still executes. The red squiggles in the editor and `npm run check` are what actually tell you. This surprises people coming from `tsc`-based setups.

**Strict mode is on**, including `noUncheckedIndexedAccess` — so `rows[0]` has type `T | undefined` and TypeScript makes you handle the empty case. It's stricter than most tutorials. That's deliberate: it catches the class of bug that's worth catching, and turning it off later is one line in `tsconfig.json`.

**JS and TS in notebooks too.** JupyterLab has three kernels — Python, JavaScript, TypeScript. Pick one when you create a notebook and your cells are that language. Handy for trying a snippet without making a file.

---

## 6. Seed the toy shop database

Guide 1 works against six rows in a table called `raw_sales`. Run this in the IDE terminal (or a notebook cell):

```bash
python - <<'PY'
import duckdb
con = duckdb.connect("/work/shop.duckdb")
con.execute("DROP TABLE IF EXISTS raw_sales")
con.execute("""
CREATE TABLE raw_sales (
  receipt_no   VARCHAR,
  sold_at      TIMESTAMP,
  kid_name     VARCHAR,
  kid_city     VARCHAR,
  toy_name     VARCHAR,
  toy_category VARCHAR,
  toy_price    DECIMAL(10,2),
  quantity     INTEGER,
  shop_name    VARCHAR
)""")
con.execute("""
INSERT INTO raw_sales VALUES
 ('R-001', TIMESTAMP '2026-08-01 10:15:00', 'Ana',  'Brampton',    'Red Robot',   'Robots',  24.99, 2, 'Main Street'),
 ('R-001', TIMESTAMP '2026-08-01 10:15:00', 'Ana',  'Brampton',    'Puzzle Cube', 'Puzzles',  9.50, 1, 'Main Street'),
 ('R-002', TIMESTAMP '2026-08-01 14:40:00', 'Ben',  'Toronto',     'Red Robot',   'Robots',  24.99, 1, 'Main Street'),
 ('R-003', TIMESTAMP '2026-08-02 11:05:00', 'Cara', 'Toronto',     'Kite',        'Outdoor', 14.00, 3, 'Lakeside'),
 ('R-004', TIMESTAMP '2026-08-02 16:20:00', 'Ana',  'Brampton',    'Puzzle Cube', 'Puzzles',  9.50, 4, 'Lakeside'),
 ('R-005', TIMESTAMP '2026-08-03 09:30:00', 'Ben',  'Mississauga', 'Red Robot',   'Robots',  29.99, 1, 'Lakeside')
""")
print(con.sql("SELECT count(*) AS lines, count(DISTINCT receipt_no) AS sales, sum(quantity*toy_price) AS revenue FROM raw_sales"))
PY
```

### Check it

```
┌───────┬───────┬─────────┐
│ lines │ sales │ revenue │
│   6   │   5   │ 194.46  │
└───────┴───────┴─────────┘
```

**`lines` is 6 but `sales` is 5.** That gap isn't a mistake in the data — it's the point of guide 1's first two concepts, and everything else there grows out of it.

Two other details, because guide 1 spends whole sections on them:

- The **Red Robot costs $24.99 on 1–2 August and $29.99 on 3 August**. A price that changed.
- **Ben appears with two different cities.** Toronto on R-002, Mississauga on R-005.

> **Tip.** In a notebook, `duckdb.connect('/work/shop.duckdb').sql(q).df()` returns a pandas DataFrame, which renders as a proper table. Far easier to read than shell output while you work through guide 1.

---

## 7. Create the etl-practice project

Guide 3 tests a project with a specific shape. Create it once:

```bash
mkdir -p /work/etl-practice/etl /work/etl-practice/tests
cd /work/etl-practice
touch etl/__init__.py etl/parsing.py etl/transforms.py etl/spark_jobs.py
touch tests/conftest.py

cat > pyproject.toml <<'TOML'
[project]
name = "etl-practice"
version = "0.1.0"
requires-python = ">=3.11"

[build-system]
requires = ["setuptools>=68"]
build-backend = "setuptools.build_meta"

[tool.pytest.ini_options]
testpaths        = ["tests"]
python_files     = ["test_*.py"]
python_functions = ["test_*"]
addopts          = "-v --strict-markers"
markers = [
  "slow: takes more than a second",
  "spark: needs a SparkSession",
  "integration: touches a real external system",
]
TOML

pip install -e .
```

The result:

```
etl-practice/
├── pyproject.toml
├── etl/
│   ├── __init__.py          ← empty, but imports break confusingly without it
│   ├── parsing.py
│   ├── transforms.py
│   └── spark_jobs.py
└── tests/
    └── conftest.py          ← shared fixtures live here
```

### Check it

```bash
pytest
# collected 0 items  ← correct, there are no tests yet
python -c "import etl; print('importable')"
```

Then open the IDE's testing sidebar. Once guide 3 has you writing tests, they appear there and run on a click.

> **Trap.** `--strict-markers` turns a **typo'd marker into an error** rather than something that silently does nothing. `@pytest.mark.slwo` would otherwise skip nothing and tell you nothing. Leave it on.

---

## 8. The warehouse and dbt

Guides 5 and 6 build a warehouse from three CSV files that the container creates on first start:

```
warehouse/landing/sales_2026-08-01.csv     3 lines
warehouse/landing/sales_2026-08-02.csv     2 lines
warehouse/landing/sales_2026-08-03.csv     2 lines  ← one is a re-sent copy of R-002
                                           ───────
                                           7 lines, 6 real sale lines
```

That deliberate duplicate is what makes deduplication, idempotency and late-arriving data real rather than theoretical. Both guides depend on it.

### Check it

```bash
python -c "
import duckdb
print(duckdb.connect().sql('''
  SELECT count(*) AS lines,
         count(DISTINCT (receipt_no, toy_name)) AS real_lines
  FROM read_csv(\"/work/warehouse/landing/*.csv\", header=true)'''))"
# 7 | 6
```

**dbt is already installed and already configured to find its profile.** `DBT_PROFILES_DIR` points at `/work/dbt`, so you never pass `--profiles-dir`:

```bash
dbt --version          # should list duckdb under "Plugins"
```

Guide 6 has you write `/work/dbt/profiles.yml` and the project itself — that's concept 2, and doing it by hand once is worth more than having it appear.

> **Trap.** `dbt docs serve` binds to `127.0.0.1` inside the container, where your browser can't reach it. Run it as `dbt docs serve --port 8080 --host 0.0.0.0`; port 8080 is published for exactly this.

The warehouse file itself, `warehouse.duckdb`, gets created by the guides. Delete it any time you want to start over — the landing CSVs are the source of truth and they're regenerated if you delete those too.

---

## 9. Daily commands

On your laptop, in the folder with `docker-compose.yml`:

| Command | What it does |
|---|---|
| `docker compose up -d` | Start it. IDE and notebooks come up with it. |
| `docker compose restart lab` | **Pick up new libraries** from `requirements.txt` / `package.json` |
| `docker compose exec lab bash` | A shell inside |
| `docker compose logs -f lab` | Watch startup, or find out why something didn't come up |
| `docker compose stop` | Stop, keep everything |
| `docker compose down` | Remove the container, **keep** installed tools and IDE settings |
| `docker compose down -v` | Also delete the tool volumes — clean slate, one rebuild away |

**None of these touch your work.** Files, notebooks and `.duckdb` databases live in your folder on your laptop the whole time.

---

## 10. Troubleshooting

| Symptom | Cause and fix |
|---|---|
| `Cannot connect to the Docker daemon` | Docker Desktop isn't running. Start it and wait. |
| `localhost:8443` won't load | Startup isn't finished. `docker compose logs -f lab`, wait for `READY`. |
| New library still not importable | The restart hasn't finished, or the name is misspelled. Check the logs for a pip error, then reload the browser tab. |
| `pip install` failed on restart | One bad line in `requirements.txt` stops the whole install. The log names it — fix that line, restart again. |
| Container exits immediately | Read the logs. Usually a network blip during install; `docker compose restart lab` retries. |
| `java: command not found` | First-run install didn't finish. Restart, watch for `READY`. |
| PySpark hangs on start | `SPARK_LOCAL_IP=127.0.0.1` is already set for exactly this. If it persists, restart. |
| `ModuleNotFoundError: etl` | You skipped `pip install -e .`, or you're not in `/work/etl-practice`. |
| Notebook has no TypeScript kernel | `tslab install --python=/opt/venv/bin/python`, then reload the tab. |
| An IDE extension didn't install | Not all VS Code extensions exist on Open VSX, which is what code-server uses. The log says which one was skipped. |
| `port is already allocated` | Something else holds it. Change the left number in `ports:` — `127.0.0.1:8444:8443`. |
| Dev server unreachable in the browser | It bound to `127.0.0.1` inside the container. Restart it with `--host 0.0.0.0`. |
| IDE lost your settings | You ran `down -v`. Settings live in the `ide-state` volume; plain `down` keeps them. |
| `Could not find profile named 'toyshop'` | `profile:` in `dbt_project.yml` must match the top key in `profiles.yml`. Usually a typo in one of the two. |
| `dbt docs serve` unreachable | Add `--host 0.0.0.0 --port 8080`. |
| Everything is subtly broken | `docker compose down -v && docker compose up -d`. Genuinely a clean slate, and your files are fine. |

---

## Where to go next

| Guide | Covers |
|---|---|
| [01 — Data Modelling](01-data-modelling.md) | Grain, facts, dimensions, keys, star schema, SCD2 |
| [02 — Async, Regex, DE Patterns](02-python-async-regex-patterns.md) | Concurrency, pattern matching, and everyday Python for pipelines |
| [03 — pytest for ETL](03-pytest-for-etl.md) | Fixtures, mocking, Spark tests, and the bugs they catch |
| [04 — Foundry OSDK & Marketplace](04-foundry-osdk-marketplace.md) | Run this one in your own Foundry workspace |
| [05 — Medallion Architecture](05-medallion-architecture.md) | Bronze, silver, gold: building a warehouse in layers |
| [06 — dbt Basics](06-dbt-basics.md) | models, ref, tests, seeds, snapshots, lineage |

Guide 1 first if you're doing all of them — guide 3's levels on fan-out and SCD2 assume its concepts 9 and 10.
