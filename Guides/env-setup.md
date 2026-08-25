# Setup

Follow these 12 steps in order. After each one there is a **You should see** box. If what you see matches, go to the next step. If it does not match, go to [Step 12](#step-12--if-something-goes-wrong).

You only do this once. It takes about 15 minutes.

---

## What you are building

One Docker container that holds everything you need:

```
   YOUR COMPUTER                    THE CONTAINER
   ┌──────────────────┐         ┌──────────────────────────────┐
   │  your folder     │◀───────▶│  Python   Node   TypeScript  │
   │  (your files)    │  shared │  Spark    dbt    DuckDB      │
   └──────────────────┘         │  an editor you open in your  │
                                │  web browser                 │
                                └──────────────────────────────┘
```

You do not install Python, Node or Java on your computer. Only Docker.

**Your files always stay on your computer.** The container just reads them. Nothing in this guide can delete your work.

---

## Steps

| Step | What you do |
|---|---|
| [1](#step-1--install-docker) | Install Docker |
| [2](#step-2--put-the-files-in-one-folder) | Put the files in one folder |
| [3](#step-3--start-the-container) | Start the container |
| [4](#step-4--wait-for-the-ready-message) | Wait for the READY message |
| [5](#step-5--open-the-editor-in-your-browser) | Open the editor in your browser |
| [6](#step-6--open-a-terminal-inside-the-editor) | Open a terminal inside the editor |
| [7](#step-7--check-that-everything-is-installed) | Check that everything is installed |
| [8](#step-8--make-the-toy-shop-database) | Make the toy shop database |
| [9](#step-9--make-the-test-project-folder) | Make the test project folder |
| [10](#step-10--check-the-warehouse-files) | Check the warehouse files |
| [11](#step-11--try-typescript) | Try TypeScript |
| [12](#step-12--if-something-goes-wrong) | If something goes wrong |

---

## Step 1 — Install Docker

Go to [docker.com](https://www.docker.com/products/docker-desktop/) and install **Docker Desktop**. On Linux, install Docker Engine and the Compose plugin instead.

Start Docker Desktop and wait until it says it is running.

Then open a terminal on your computer and type:

```bash
docker --version
docker compose version
```

**You should see** two version numbers, like this:

```
Docker version 27.3.1, build ce12230
Docker Compose version v2.29.7
```

**If `docker compose version` gives an error**, try `docker-compose --version` instead. If that works, you have an older Docker. Everything below still works — just type `docker-compose` where this guide says `docker compose`.

---

## Step 2 — Put the files in one folder

Make a folder anywhere you like. Put all 10 files inside it:

```
de-practice/
├── docker-compose.yml       ← starts everything. you will not edit this.
├── requirements.txt         ← list of Python libraries
├── package.json             ← list of JavaScript libraries
├── 00-setup.md              ← this file
├── 01-data-modelling.md
├── 02-python-async-regex-patterns.md
├── 03-pytest-for-etl.md
├── 04-foundry-osdk-marketplace.md
├── 05-medallion-architecture.md
└── 06-dbt-basics.md
```

Now go into that folder in your terminal:

```bash
cd path/to/de-practice
ls
```

**You should see** all 10 file names listed.

**This folder matters.** Every `docker compose` command in this guide must be typed while you are inside it. If a command says "not found", check where you are with `pwd`.

---

## Step 3 — Start the container

Type this:

```bash
docker compose up -d
```

**You should see** something like:

```
[+] Running 2/2
 ✔ Network de-practice_default  Created
 ✔ Container de-lab             Started
```

This means Docker started it. It does **not** mean it is ready yet — that is the next step.

**The first time is slow.** Docker downloads Python, then installs Java, Node, dbt, the editor and all the libraries. Expect **5 to 8 minutes**. Every time after this, it takes about 10 seconds.

---

## Step 4 — Wait for the READY message

Type this to watch what it is doing:

```bash
docker compose logs -f lab
```

Lines will scroll past — installing Java, installing Python libraries, and so on. This is normal. Wait.

**You should see** this at the end:

```
[lab] READY
        IDE        http://localhost:8443
        Notebooks  http://localhost:8888
        Shell      docker compose exec lab bash
```

When you see `READY`, press **Ctrl-C**. That stops the scrolling text. It does **not** stop the container.

**If it stops before READY** and shows a red error, read the last few lines. It is usually an internet problem during download. Type `docker compose restart lab` and it will try again.

---

## Step 5 — Open the editor in your browser

Open your web browser and go to:

```
http://localhost:8443
```

**You should see** Visual Studio Code, running inside your browser. On the left is a file list showing your 10 files.

This is where you will do most of your work. It is a real code editor — not a website that looks like one.

**Nobody else can open this.** It is locked to your own computer, so there is no password. It will not work from your phone or another machine, and that is on purpose.

---

## Step 6 — Open a terminal inside the editor

In the editor, click the menu: **Terminal → New Terminal**.

A black box opens at the bottom of the screen.

**You should see** a prompt that ends like this:

```
root@de-lab:/work#
```

Two things to notice:

- `de-lab` means you are **inside the container**, not on your own computer.
- `/work` is your folder. The same files, seen from inside.

**From here on, every command in this guide and in all six guides goes in this box.** Not in your computer's own terminal.

If you prefer your own terminal, type `docker compose exec lab bash` there instead. It is the same thing.

---

## Step 7 — Check that everything is installed

Type these four commands, one at a time, in the editor's terminal.

**Check Python:**

```bash
python -c "import duckdb, pandas, pyspark; print('python ok')"
```

You should see: `python ok`

**Check the test tool:**

```bash
pytest --version
```

You should see: `pytest 8.x.x`

**Check Java** (Spark will not start without it):

```bash
java -version
```

You should see three lines about `openjdk version`.

**Check Node, TypeScript and dbt:**

```bash
node --version && tsc --version && dbt --version
```

You should see a version number for each. The dbt output should list `duckdb` under `Plugins`.

**All four worked?** The hard part is done.

---

## Step 8 — Make the toy shop database

Guide 1 needs a small database. Copy this whole block into the terminal and press Enter:

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

**You should see** exactly this:

```
┌───────┬───────┬─────────┐
│ lines │ sales │ revenue │
│   6   │   5   │ 194.46  │
└───────┴───────┴─────────┘
```

**Look at those numbers.** There are 6 rows but only 5 sales. That is not a mistake. One shopping trip put two toys on one receipt, so it takes up two rows. Guide 1 is mostly about what to do with that.

A new file called `shop.duckdb` has now appeared in your folder. You will see it in the editor's file list too.

---

## Step 9 — Make the test project folder

Guide 3 needs a small project with a set shape. Copy this whole block in:

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

Now check it worked:

```bash
pytest
```

**You should see:**

```
collected 0 items
```

**Zero is the right answer.** You have not written any tests yet. If it says `collected 0 items` without an error, the project is set up correctly.

One more check:

```bash
python -c "import etl; print('importable')"
```

You should see: `importable`

Then go back to the main folder:

```bash
cd /work
```

---

## Step 10 — Check the warehouse files

Guides 5 and 6 use three small CSV files. The container already made them for you in Step 3. Just check they are there:

```bash
ls /work/warehouse/landing/
```

**You should see:**

```
sales_2026-08-01.csv  sales_2026-08-02.csv  sales_2026-08-03.csv
```

Now count what is inside them:

```bash
python -c "
import duckdb
print(duckdb.connect().sql('''
  SELECT count(*) AS lines,
         count(DISTINCT (receipt_no, toy_name)) AS real_lines
  FROM read_csv(\"/work/warehouse/landing/*.csv\", header=true)'''))"
```

**You should see:**

```
┌───────┬────────────┐
│ lines │ real_lines │
│   7   │     6      │
└───────┴────────────┘
```

**7 lines but only 6 real ones.** The shop's till sent one sale twice — once on the first day, and again by mistake on the third day. Guides 5 and 6 teach you how to deal with that. It is put there on purpose, so do not "fix" it.

---

## Step 11 — Try TypeScript

The container made a small TypeScript project for you too. Run it:

```bash
cd /work
npm run dev
```

**You should see:**

```
[ 'ANA', 'BEN' ]
```

Now leave that running. In the editor, open the file `ts/index.ts` from the file list, change one of the names, and save with **Ctrl-S**.

**You should see** the terminal print the new result by itself, without you typing anything. That is watch mode. It is the fastest way to practise.

Press **Ctrl-C** to stop it.

Four commands worth remembering:

| Command | What it does |
|---|---|
| `npm run dev` | Runs your file again every time you save |
| `npm run check` | Checks your types and tells you about mistakes |
| `tsx myfile.ts` | Runs one TypeScript file right now |
| `npm run serve` | Starts a web server at `http://localhost:5173` |

**One thing that surprises people:** `npm run dev` runs your file even if the types are wrong. It ignores type mistakes. Use `npm run check` when you want to be told about them.

---

## Step 12 — If something goes wrong

Find your problem in the left column.

| What you see | What to do |
|---|---|
| `Cannot connect to the Docker daemon` | Docker Desktop is not running. Open it, wait, try again. |
| `localhost:8443` does not open | It is not ready yet. Run `docker compose logs -f lab` and wait for `READY`. |
| The container stops on its own | Run `docker compose logs lab` and read the last lines. Usually the internet dropped during download. Run `docker compose restart lab`. |
| `java: command not found` | The install did not finish. Run `docker compose restart lab` and wait for `READY`. |
| `no such file or directory` on a `docker compose` command | You are in the wrong folder on your computer. Go back to the folder holding `docker-compose.yml`. |
| `ModuleNotFoundError: etl` | You are not in `/work/etl-practice`, or you skipped `pip install -e .` in Step 9. |
| `port is already allocated` | Another program is using that port. Open `docker-compose.yml`, find `8443:8443`, change the **left** number to `8444`. Then `docker compose up -d`. |
| A web page in the container will not open in your browser | The program is only listening to itself. Start it again with `--host 0.0.0.0` at the end. |
| Spark hangs and never starts | Run `docker compose restart lab`. |
| Nothing works and you want to start over | Run `docker compose down -v` then `docker compose up -d`. Takes 5 minutes. **Your files are safe** — this only deletes installed programs. |

**Still stuck?** Run `docker compose logs lab` and read the last 20 lines. The real error is almost always there, near the bottom.

---

## Adding a library later

You will need this often, so it has its own short section.

**For a Python library:**

1. Open `requirements.txt` in the editor.
2. Add the name on its own line at the bottom, for example `polars`.
3. Save the file.
4. In your computer's terminal, run `docker compose restart lab`.
5. Wait about 20 seconds, then reload the browser tab.

**You should see** that `import polars` now works in the terminal.

**For a JavaScript library**, it is one command in the editor's terminal:

```bash
npm install zod
```

That installs it **and** adds it to `package.json` for you.

**Your work is never touched by this.** Files, notebooks and databases all stay exactly as you left them.

> There is a shortcut: typing `pip install polars` works right away. But it disappears the next time you rebuild. Use it to try something out. Put it in `requirements.txt` once you decide to keep it.

---

## Commands you will use every day

Type these in **your computer's** terminal, in the folder with `docker-compose.yml`:

| Command | What it does |
|---|---|
| `docker compose up -d` | Start everything |
| `docker compose restart lab` | Pick up new libraries you added |
| `docker compose logs -f lab` | Watch what it is doing |
| `docker compose stop` | Stop it for now, keep everything |
| `docker compose down -v` | Delete installed programs and start fresh |

And these in **the editor's** terminal:

| Command | What it does |
|---|---|
| `pytest` | Run your tests |
| `dbt build` | Run and test your dbt models |
| `npm run dev` | Run your TypeScript, re-running on save |

---

## Three places to work

They all show the same files. Pick whichever suits what you are doing.

| Where | Address | Best for |
|---|---|---|
| **Editor** | `http://localhost:8443` | Most things. Writing files, running tests, notebooks. |
| **Notebooks** | `http://localhost:8888` | Trying one small idea. Python, JavaScript or TypeScript. |
| **dbt docs** | `http://localhost:8080` | Guide 6 only, after you run `dbt docs serve --host 0.0.0.0 --port 8080` |

---

## What to read next

| Guide | What it covers |
|---|---|
| [01 — Data Modelling](01-data-modelling.md) | How to organise data into tables that make sense |
| [02 — Async, Regex, DE Patterns](02-python-async-regex-patterns.md) | Python for doing many things at once, and finding patterns in text |
| [03 — pytest for ETL](03-pytest-for-etl.md) | Writing tests that catch real data bugs |
| [04 — Foundry OSDK & Marketplace](04-foundry-osdk-marketplace.md) | Do this one in your own Foundry workspace, not here |
| [05 — Medallion Architecture](05-medallion-architecture.md) | Building a warehouse in three layers |
| [06 — dbt Basics](06-dbt-basics.md) | The same warehouse again, built with dbt |

**Start with guide 1.** Guides 3, 5 and 6 all use words that guide 1 explains.
