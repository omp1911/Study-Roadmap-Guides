# Python Async, Regex & Data Engineering Patterns
### One-file practice guide — setup, lessons, drills, and solutions

Everything you need is in this document. Nothing else to download.

---

## Contents

**[Section 0 — Environment Setup](#section-0--environment-setup)**
Free cloud dev environment via GitHub Codespaces. All config files inline, copy-paste. ~10 minutes, once.

**[Section 1 — Asynchronous Python](#part-1--asynchronous-python)**
Why async exists · coroutines and `await` · `gather` vs `TaskGroup` · the blocking trap · error handling · timeouts and cancellation · rate limiting · queues and backpressure · async generators · when *not* to use async. **12 drills.**

**[Section 2 — Regex](#part-2--regex)**
Building blocks · named groups · the five functions · greedy vs lazy · lookarounds · real pipeline patterns (log lines, S3 paths, column cleaning, row validation) · the two traps. **10 drills.**

**[Section 3 — Data Engineering Patterns](#part-3--python-patterns-for-data-engineering)**
Generators for huge files · batching · context managers · dataclasses · retry with backoff · logging · caching. **11 drills.**

**[Section 4 — Solutions](#section-4--solutions)**
Full worked answers to all 33 drills. Don't open early.

---

## How to use this

1. Do **Section 0** first. Ten minutes, and you'll have a working environment that survives between sessions.
2. Read a lesson section, then **type the example out by hand** and run it. Don't copy-paste. Typing is half the learning.
3. Do that section's drills before moving on.
4. Give each drill fifteen honest minutes before looking at Section 4.

**Suggested reading order:** Section 1 (1.1–1.6) → Section 2 (regex is quick) → Section 1 (1.7–1.11) → Section 3.

A nine-day schedule is at the end of Section 3.

---

# SECTION 0 — ENVIRONMENT SETUP

One file. Work top to bottom. About 10 minutes, once.

Every file's full contents are in a code block below. Copy each exactly as shown.

---

### What you're setting up

**GitHub Codespaces** — a full dev environment in a browser tab. Nothing installed on your laptop, files persist between sessions, you can stop and resume anytime. Free tier for personal accounts is 120 core-hours/month (about 60 real hours on the default 2-core machine) plus 15 GB storage.

Same deal as your `js-ts-practice` repo. The `.devcontainer/devcontainer.json` **is a Docker config** — if you ever want this locally, Docker Desktop runs the identical environment with no changes.

**Free-forever backup:** Google Cloud Shell — 50 hrs/week, 5 GB persistent home, Python preinstalled, no card. Use it if you burn Codespaces hours mid-month.

---

### Step 1 — Create the repo

On github.com: **New repository** → name it `python-async-practice` → check "Add a README" → **Create**.

---

### Step 2 — Add the config files

In the repo: **Add file → Create new file**. Typing `.devcontainer/devcontainer.json` as the filename creates the folder automatically.

#### `.devcontainer/devcontainer.json`

```json
{
  "name": "python-async-practice",
  "image": "mcr.microsoft.com/devcontainers/python:3.12",
  "postCreateCommand": "pip install --user -r requirements.txt",
  "customizations": {
    "vscode": {
      "extensions": [
        "ms-python.python",
        "ms-python.vscode-pylance",
        "charliermarsh.ruff"
      ],
      "settings": {
        "python.testing.pytestEnabled": true,
        "editor.formatOnSave": true,
        "editor.defaultFormatter": "charliermarsh.ruff"
      }
    }
  }
}
```

#### `requirements.txt`

```
pytest>=8.3
pytest-asyncio>=0.24
httpx>=0.27
aiosqlite>=0.20
ruff>=0.7
```

Deliberately small. Nearly every drill uses only the standard library — `asyncio`, `re`, `itertools`, `dataclasses`. That's on purpose: the standard library is what you'll actually reach for at work, and it never breaks.

#### `pyproject.toml`

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
asyncio_default_fixture_loop_scope = "function"
testpaths = ["."]

[tool.ruff]
line-length = 100
target-version = "py312"
```

`asyncio_mode = "auto"` is the important line. Without it, every async test needs a `@pytest.mark.asyncio` decorator and you'll spend an hour confused about why your tests "pass" without running.

#### `.gitignore`

```
__pycache__/
*.pyc
.pytest_cache/
.ruff_cache/
*.db
.env
venv/
```

---

### Step 3 — Launch it

Repo main page → green **Code** button → **Codespaces** tab → **Create codespace on main**.

Wait about a minute. You get VS Code in the browser with Python 3.12 and everything installed.

---

### Step 4 — Create the folders

In the Codespace terminal:

```bash
mkdir -p part1_async part2_regex part3_de_patterns solutions
touch part1_async/.gitkeep part2_regex/.gitkeep part3_de_patterns/.gitkeep
```

Save this document as `GUIDE.md` in the repo root so you can read it inside the Codespace, side by side with your code.

---

### Step 5 — Prove it works

Create `part1_async/hello.py`:

```python
import asyncio
import time


async def slow_thing(name: str, seconds: float) -> str:
    await asyncio.sleep(seconds)
    return f"{name} finished"


async def main() -> None:
    start = time.perf_counter()
    results = await asyncio.gather(
        slow_thing("query_a", 1),
        slow_thing("query_b", 1),
        slow_thing("query_c", 1),
    )
    print(results)
    print(f"took {time.perf_counter() - start:.2f}s")


asyncio.run(main())
```

Run it:

```bash
python part1_async/hello.py
```

You should see about **1.0 seconds**, not 3.0. Three one-second waits happened at the same time. If you see 3.0, something's wrong — tell me.

---

### Step 6 — Prove tests work

Create `part1_async/test_hello.py`:

```python
import asyncio


async def double(x: int) -> int:
    await asyncio.sleep(0.01)
    return x * 2


async def test_double():
    assert await double(3) == 6
```

Run:

```bash
pytest -q
```

`1 passed`. You're set up.

---

### Daily commands

```bash
python part1_async/whatever.py    # run one file
pytest -q                         # run all tests
pytest part1_async -q             # run one folder's tests
ruff check .                      # find problems
ruff format .                     # auto-format
```

---

### Saving your work

Codespaces persist, but push anyway — it's free insurance and keeps the Git muscle warm:

```bash
git add -A
git commit -m "async drills part 1"
git push
```

---

### Two gotchas that will bite you

**1. Never name a file after a module you're importing.** A file called `asyncio.py` or `re.py` in your folder will shadow the real library and produce baffling errors. Name things `test_asyncio_basics.py`, not `asyncio.py`.

**2. Codespaces sleep after 30 minutes idle.** Your files are safe; the machine just stops. Reopening takes a few seconds. Delete Codespaces you're done with — they eat your storage quota even while stopped.

---

# SECTIONS 1–3 — THE GUIDE

Plain English. Real data engineering examples. Every drill has a solution in **Section 4** at the bottom of this document, which you should not open until you've genuinely tried.

**How to use this:** read a section, then immediately type the example out by hand and run it. Do not copy-paste the examples. Typing them is half the learning. Then do the drills for that section before moving on.

**Suggested order:** Part 1 sections 1.1–1.6 → Part 2 (regex, it's quick) → Part 1 sections 1.7–1.11 → Part 3.

---

## PART 1 — ASYNCHRONOUS PYTHON

### 1.1 Why async exists at all

Imagine you're loading 200 files from an API into Snowflake. Each API call takes 1 second, but almost none of that second is your computer doing work. It's your computer **waiting** for the network.

Normal Python code:

```
call 1: [wait 1s] → done
call 2: [wait 1s] → done
call 3: [wait 1s] → done
...
200 seconds total, CPU sitting idle the whole time
```

Async code:

```
start all 200 calls → [wait 1s while all of them travel] → all done
about 1-2 seconds total
```

The key idea in one sentence: **async lets one program do something useful while it waits, instead of standing still.**

The waiter analogy is the one that sticks. A bad waiter takes your order, walks to the kitchen, stands there watching the chef cook, brings your food back, then finally goes to the next table. A good waiter takes your order, hands it to the kitchen, and immediately goes to the next table while your food cooks. Same one waiter. Far more tables served. The waiter never cooks faster — they just stop standing around.

**Critical limit:** async only helps with *waiting*. It does not make computation faster. If your bottleneck is a heavy pandas transform or a big sort, async does nothing for you. Section 1.11 covers what to use instead.

For a data engineer, "waiting" means: API calls, database queries, reading and writing cloud storage, message queues. Which is most of what a pipeline does.

### 1.2 The three words you need

**Coroutine** — a function defined with `async def`. Calling it does not run it. It hands you back an object that says "I'm ready to run when someone runs me."

**await** — "start this, and let other work happen until it comes back." You can only use `await` inside an `async def`.

**Event loop** — the manager. It holds all your paused tasks and wakes each one up the moment its wait is over. `asyncio.run()` starts it.

The single most common beginner mistake:

```python
async def get_data():
    await asyncio.sleep(1)
    return "rows"

result = get_data()          # WRONG — result is a coroutine object, not "rows"
result = await get_data()    # RIGHT — inside an async function
```

If you ever print something and see `<coroutine object get_data at 0x7f...>`, you forgot an `await`. That's the whole bug, every time.

### 1.3 Your first real one

```python
import asyncio
import time


async def fetch_table(name: str, seconds: float) -> dict:
    """Pretend this is a real API call or database query."""
    print(f"  starting {name}")
    await asyncio.sleep(seconds)      # the "waiting" part
    print(f"  finished {name}")
    return {"table": name, "rows": 1000}


async def main() -> None:
    start = time.perf_counter()
    result = await fetch_table("orders", 1)
    print(result)
    print(f"took {time.perf_counter() - start:.2f}s")


asyncio.run(main())
```

That takes 1 second. Nothing gained yet, because there's only one thing to wait for. Async pays off the moment you have two.

### 1.4 Running many at once

Two tools. Learn both; they're used in different situations.

#### `asyncio.gather` — the simple one

```python
async def main() -> None:
    start = time.perf_counter()
    results = await asyncio.gather(
        fetch_table("orders", 1),
        fetch_table("customers", 2),
        fetch_table("payments", 1),
    )
    print(results)
    print(f"took {time.perf_counter() - start:.2f}s")
```

Takes **2 seconds**, not 4. It's as slow as the slowest one, not the sum of all of them.

Results come back **in the order you passed them in**, not the order they finished. `results[0]` is always orders. This matters more than people expect — it means you can safely zip results back against your input list.

For a list you built dynamically, use `*`:

```python
tables = ["orders", "customers", "payments", "refunds"]
results = await asyncio.gather(*(fetch_table(t, 1) for t in tables))
```

#### `asyncio.TaskGroup` — the safer one (Python 3.11+)

```python
async def main() -> None:
    async with asyncio.TaskGroup() as tg:
        orders = tg.create_task(fetch_table("orders", 1))
        customers = tg.create_task(fetch_table("customers", 2))
    # everything is guaranteed finished by the time you get here
    print(orders.result(), customers.result())
```

More typing, better behaviour. The difference shows up when something fails, which is section 1.6.

**Rule of thumb:** `gather` for quick scripts and when you want results as a list. `TaskGroup` for anything going to production. If your new role is on Python 3.11+, `TaskGroup` should become your default.

### 1.5 The trap that ruins everything

This is the mistake that makes people say "I tried async and it didn't help."

```python
import time
import asyncio


async def broken(name):
    time.sleep(1)              # ← BLOCKING. Freezes the entire program.
    return name


async def working(name):
    await asyncio.sleep(1)     # ← Yields control. Other tasks run.
    return name
```

`time.sleep()` stops **everything**. Not just this task. Every other task in your program freezes too, because they all share one thread. Run `gather` over three `broken()` calls and you get 3 seconds. Async gained you nothing.

Anything that blocks does this, and most libraries you already use are blocking:

| Blocking (freezes the loop) | Async-safe |
|---|---|
| `time.sleep()` | `await asyncio.sleep()` |
| `requests.get()` | `await httpx.AsyncClient().get()` |
| `open()` / `f.read()` | `await aiofiles.open()` |
| `psycopg2`, `snowflake-connector-python` | `asyncpg`, or the escape hatch below |
| `pd.read_csv()` on a huge file | the escape hatch below |

**The escape hatch:** when you must call blocking code, push it to a separate thread so the loop keeps running.

```python
# Modern and simple (Python 3.9+)
rows = await asyncio.to_thread(snowflake_cursor.execute, "SELECT * FROM orders")

# The older form you'll see in existing codebases
loop = asyncio.get_running_loop()
rows = await loop.run_in_executor(None, blocking_function, arg1, arg2)
```

`asyncio.to_thread` is the one to reach for. Remember it — Snowflake's Python connector is synchronous, so you will need this in your new role.

**How to spot the bug:** if your "async" code takes exactly as long as the sequential version, you have a blocking call hiding in there. That symptom has one cause.

### 1.6 When things fail

Pipelines fail constantly. This section is the difference between a script and a job.

#### `gather` default behaviour: one failure kills the batch

```python
results = await asyncio.gather(fetch("a"), broken_fetch("b"), fetch("c"))
# raises immediately. You lose the results of a and c.
```

#### `return_exceptions=True`: collect everything, sort it out after

```python
results = await asyncio.gather(
    fetch("a"), broken_fetch("b"), fetch("c"),
    return_exceptions=True,
)
# ['a done', ValueError('bad row'), 'c done']

good = [r for r in results if not isinstance(r, Exception)]
bad = [r for r in results if isinstance(r, Exception)]
print(f"loaded {len(good)} tables, {len(bad)} failed")
```

This is the pattern you want for most ingestion work. Load the 47 files that worked, log the 3 that didn't, don't throw away good data because one file was malformed.

Pair the results back to their inputs by position, since order is guaranteed:

```python
for table, result in zip(tables, results):
    if isinstance(result, Exception):
        logger.error("table %s failed: %s", table, result)
```

#### `TaskGroup` behaviour: fail fast and clean

```python
try:
    async with asyncio.TaskGroup() as tg:
        for t in tables:
            tg.create_task(load_table(t))
except* ValueError as eg:
    print(f"{len(eg.exceptions)} tables had bad data")
```

If one task raises, `TaskGroup` **cancels the rest** and then raises an `ExceptionGroup` containing everything that went wrong. That `except*` syntax (note the star) is for catching from a group.

This is the right shape when the tasks are all-or-nothing — a set of writes that must land together, or a load where partial data is worse than no data.

**Choosing between them:** `gather(return_exceptions=True)` when partial success is useful. `TaskGroup` when partial success is dangerous.

### 1.7 Timeouts and cancellation

A hung API call with no timeout will hang your pipeline forever. Airflow will show the task as running at 3am and nobody will know why.

```python
try:
    async with asyncio.timeout(5):
        data = await slow_api_call()
except TimeoutError:
    logger.warning("api timed out after 5s, using cached data")
    data = cached
```

`asyncio.timeout` is a context manager (Python 3.11+). It covers everything inside the block, including several awaits.

Older form, still everywhere:

```python
try:
    data = await asyncio.wait_for(slow_api_call(), timeout=5)
except TimeoutError:
    ...
```

**Cleanup when cancelled.** When a task is cancelled, Python raises `CancelledError` inside it. Use `finally` to release resources:

```python
async def load_with_cleanup(conn):
    try:
        await conn.execute("BEGIN")
        await do_work(conn)
        await conn.execute("COMMIT")
    finally:
        await conn.close()      # runs even on cancellation
```

**Do not swallow `CancelledError`.** If you catch it, re-raise it. Catching it silently produces tasks that refuse to die, and shutdowns that hang.

```python
except asyncio.CancelledError:
    logger.info("cancelled, cleaning up")
    raise            # ← this line is not optional
```

### 1.8 Not overwhelming the thing you're calling

Firing 5,000 concurrent requests at an API gets you rate-limited, or gets your IP blocked, or takes down an internal service. You need a cap.

`Semaphore` is a bouncer holding a fixed number of tickets. Take one to enter, give it back on the way out.

```python
async def fetch_limited(sem: asyncio.Semaphore, url: str):
    async with sem:                  # wait here if all tickets are taken
        return await fetch(url)


async def main():
    sem = asyncio.Semaphore(10)      # max 10 in flight at any moment
    urls = [f"https://api.example.com/page/{i}" for i in range(5000)]
    results = await asyncio.gather(
        *(fetch_limited(sem, u) for u in urls),
        return_exceptions=True,
    )
```

All 5,000 tasks get created, but only 10 are ever actually running. The rest wait politely.

**Picking the number:** start at 10. If the API publishes a rate limit, stay under it. Higher is not always faster — past a certain point you just build a queue on the server and latency gets worse for everyone. This is a number you tune by measuring, not by guessing.

### 1.9 Queues — producer and consumer

The shape of a real streaming pipeline: one thing reads, several things process, and the reader doesn't get miles ahead of the writers.

```python
async def producer(queue: asyncio.Queue, n_workers: int):
    for record in read_source():
        await queue.put(record)
    for _ in range(n_workers):
        await queue.put(None)          # one poison pill per worker


async def worker(queue: asyncio.Queue, name: str):
    while True:
        record = await queue.get()
        if record is None:             # my signal to stop
            queue.task_done()
            break
        await transform_and_write(record)
        queue.task_done()


async def main():
    n = 5
    queue = asyncio.Queue(maxsize=100)     # ← backpressure
    await asyncio.gather(
        producer(queue, n),
        *(worker(queue, f"w{i}") for i in range(n)),
    )
```

Two things worth understanding here:

**`maxsize=100` is backpressure.** Once 100 records are waiting, `await queue.put()` pauses the producer until a worker takes something off. Without a maxsize, a fast producer reading a 50 GB file will pull the whole thing into memory while slow workers fall behind. This one argument prevents a very common out-of-memory failure.

**The `None` values are "poison pills."** Workers loop forever by design, so you need a way to tell them the work is finished. One `None` per worker, because each worker consumes exactly one and then exits.

### 1.10 Async generators — streaming and pagination

An async generator produces values over time instead of all at once. Perfect for paginated APIs, where you don't know the page count up front and don't want all pages in memory.

```python
async def fetch_all_pages(client, url: str):
    page = 1
    while True:
        response = await client.get(url, params={"page": page})
        data = response.json()
        if not data["results"]:
            return
        yield data["results"]          # hand back one page, then pause
        page += 1


async def main():
    async with httpx.AsyncClient() as client:
        async for batch in fetch_all_pages(client, "https://api.example.com/orders"):
            await write_to_warehouse(batch)
```

Note `async for` rather than `for`. Memory stays flat regardless of whether there are 3 pages or 30,000, because only one page exists at a time.

### 1.11 When NOT to use async

Worth knowing cold, because using the wrong one is a classic interview question and a classic production mistake.

Python has a **GIL** (Global Interpreter Lock): only one thread runs Python bytecode at a time in a single process. So:

| Your bottleneck | Use | Why |
|---|---|---|
| Network / API / DB / disk waits | **asyncio** | Thousands of concurrent waits, one thread, low overhead |
| Same, but the library is blocking and you can't change it | **threads** (`to_thread`, `ThreadPoolExecutor`) | Threads release the GIL during I/O waits |
| Heavy computation — parsing, math, big transforms | **multiprocessing** | Separate processes, separate GILs, real parallelism |
| Genuinely huge data | **Spark / Snowflake / DuckDB** | Don't hand-roll distribution in Python |

Said plainly: **asyncio for waiting, multiprocessing for thinking.** If you're calling 500 APIs, async. If you're parsing 500 large XML files, multiprocessing. If you're doing both, async for the fetch and hand the parsing to a process pool.

Coming from your background, the honest framing is: async is for the orchestration layer of a pipeline. The actual heavy transform belongs in the warehouse.

---

### Part 1 drills

Write each in `part1_async/`. Try each for at least fifteen minutes before opening the solutions.

**1.** Write `fetch_table(name, seconds)` that sleeps and returns `{"table": name, "rows": 100}`. Call it for five tables sequentially with a `for` loop, then with `gather`. Print elapsed time for each. Confirm the difference.

**2.** Predict the output order of this before you run it. Then run it. If your prediction was wrong, work out why — this drill is worth more than the rest combined.

```python
import asyncio

async def a():
    print("a start")
    await asyncio.sleep(0)
    print("a end")

async def b():
    print("b start")
    await asyncio.sleep(0.1)
    print("b end")

async def main():
    print("main start")
    await asyncio.gather(a(), b())
    print("main end")

asyncio.run(main())
```

**3.** Take drill 1's `gather` version and replace `asyncio.sleep` with `time.sleep`. Measure. Explain in one sentence why it got slower.

**4.** Write `load_file(name)` that fails with `ValueError` if the name contains `"corrupt"`, otherwise succeeds after 0.2s. Run it over `["a", "corrupt_b", "c", "corrupt_d", "e"]` with `gather(..., return_exceptions=True)`. Print `"loaded 3 files, 2 failed"` and log which ones failed.

**5.** Same input, but with `TaskGroup` and `except*`. Note what happens to the files that hadn't started yet.

**6.** Write `flaky_api()` that fails the first two times it's called and succeeds on the third. Wrap it in a retry helper: up to 5 attempts, waiting 1s, 2s, 4s, 8s between them (exponential backoff). Print each attempt.

**7.** Add a small random amount to each backoff delay (jitter). Then write one sentence explaining why jitter matters when 200 workers all fail at the same moment.

**8.** Write `fetch(i)` that sleeps 0.1s. Run it for 100 values with a `Semaphore(5)`. Track and print the peak number running at once. It should be exactly 5.

**9.** Build the producer/consumer from 1.9 with a real `asyncio.Queue`, 20 records and 3 workers. Print which worker handled which record. Then set `maxsize=2` and describe what changes.

**10.** Write an async generator `paginate(total_pages)` that yields a list of 3 fake rows per page with a 0.05s delay. Consume it with `async for` and count total rows without ever holding more than one page in memory.

**11.** Write `fetch_with_timeout(seconds)` using `asyncio.timeout(1)`. Call it with 0.5 and with 2.0. Handle the timeout and return `"fallback"` instead of crashing.

**12.** Combine everything: fetch 50 "tables" concurrently, capped at 8 at a time, each with a 2-second timeout, retrying twice on failure, collecting successes and failures separately, and printing a summary at the end. This is a real ingestion job in about 40 lines.

---

## PART 2 — REGEX

### 2.1 What regex actually is

A regex is a pattern that describes what text should look like. You hand it to Python and ask "does this match?" or "pull out the bits that match."

For a data engineer it earns its keep in five places: parsing log lines, validating IDs and codes, extracting fields from file paths, cleaning column names, and finding bad rows in dirty source data.

**Always use raw strings.** Write `r"\d+"`, never `"\d+"`. In a normal string, `\d` may be interpreted by Python before regex ever sees it. The `r` prefix says "hands off, pass this through untouched." Getting into this habit removes an entire category of confusing bug.

### 2.2 The building blocks

Learn these fifteen and you can read almost any pattern.

**What to match:**

| Pattern | Means | Example match |
|---|---|---|
| `\d` | any digit | `7` |
| `\w` | letter, digit, or underscore | `a`, `9`, `_` |
| `\s` | any whitespace | space, tab, newline |
| `.` | any character at all | anything |
| `[abc]` | one of a, b, or c | `b` |
| `[a-z]` | any lowercase letter | `q` |
| `[^abc]` | anything **except** a, b, c | `z` |

Capitals invert them. `\D` is "not a digit," `\S` is "not whitespace," `\W` is "not a word character."

**How many:**

| Pattern | Means |
|---|---|
| `*` | zero or more |
| `+` | one or more |
| `?` | zero or one (optional) |
| `{3}` | exactly 3 |
| `{2,5}` | between 2 and 5 |
| `{3,}` | 3 or more |

**Where:**

| Pattern | Means |
|---|---|
| `^` | start of string |
| `$` | end of string |
| `\b` | word boundary |

`\b` is the underrated one. `r"\bid\b"` matches the word "id" but not the "id" inside "invalid" or "identity."

Read a pattern left to right, out loud, in plain English:

```
r"^\d{4}-\d{2}-\d{2}$"
 ^      start of string
 \d{4}  four digits
 -      a literal dash
 \d{2}  two digits
 -      a dash
 \d{2}  two digits
 $      end of string
```

That's a date check. Anchored on both ends, so `"2026-08-22 extra junk"` correctly fails.

### 2.3 Groups — pulling pieces out

Round brackets capture a piece so you can retrieve it.

```python
import re

m = re.search(r"(\d{4})-(\d{2})-(\d{2})", "loaded on 2026-08-22 ok")
m.group(0)   # '2026-08-22'  ← the whole match
m.group(1)   # '2026'
m.groups()   # ('2026', '08', '22')
```

**Named groups are better.** `(?P<name>...)` gives each piece a label:

```python
pattern = r"(?P<year>\d{4})-(?P<month>\d{2})-(?P<day>\d{2})"
m = re.search(pattern, "2026-08-22")
m.group("year")     # '2026'
m.groupdict()       # {'year': '2026', 'month': '08', 'day': '22'}
```

`groupdict()` returning a ready-made dict is the reason to prefer named groups in pipeline code. Six months later, `m.group("job_id")` still reads clearly and `m.group(4)` doesn't. And when someone adds a bracket to the pattern, positional numbers all shift and your code silently reads the wrong field. Named groups don't have that failure mode.

Use `(?:...)` when you need brackets for grouping but don't want to capture:

```python
r"(?:https?://)(?P<host>[^/]+)"    # the protocol is grouped but not captured
```

### 2.4 The five functions

```python
import re

text = "job_id=4412 rows=0 status=FAILED"

re.search(r"rows=(\d+)", text)      # first match anywhere → Match or None
re.match(r"job_id", text)           # only at the START → Match or None
re.findall(r"(\w+)=(\S+)", text)    # every match → list
re.finditer(r"(\w+)=(\S+)", text)   # every match → iterator of Match objects
re.sub(r"\d+", "N", text)           # replace → 'job_id=N rows=N status=FAILED'
re.split(r"\s+", text)              # split → ['job_id=4412', 'rows=0', ...]
```

Two practical notes:

**`search` vs `match`.** `match` only looks at the start. Most of the time you want `search`. Reaching for `match` and getting `None` on text that obviously contains your pattern is a rite of passage.

**Compile patterns you reuse.** Inside a loop over a million log lines, compile once:

```python
LOG_PATTERN = re.compile(r"^(?P<ts>\S+ \S+) (?P<level>\w+) \[(?P<job>[\w-]+)\]")

for line in log_file:
    m = LOG_PATTERN.search(line)
    if m:
        yield m.groupdict()
```

Python caches recent patterns so the gain is smaller than people claim, but naming your patterns as module-level constants makes the code far more readable, and that's the real win.

**Always check for `None`.** `re.search` returns `None` when there's no match, and `None.group()` is an `AttributeError` at 3am.

```python
m = pattern.search(line)
if m is None:
    logger.warning("unparseable line: %s", line[:100])
    continue
```

### 2.5 Greedy vs lazy — the classic trap

By default, quantifiers grab as much as they possibly can.

```python
text = 'msg="first" other msg="second"'

re.search(r'msg="(.*)"', text).group(1)
# 'first" other msg="second'   ← took everything to the LAST quote

re.search(r'msg="(.*?)"', text).group(1)
# 'first'                      ← the ? makes it stop at the first quote
```

The `?` after `*` or `+` means "take as little as possible." `.*?` and `.+?` are the lazy versions.

There's usually a better fix than lazy matching, though: say what you actually mean.

```python
re.search(r'msg="([^"]*)"', text).group(1)   # 'first'
```

`[^"]*` means "any characters that aren't a quote." It's clearer than `.*?`, and much faster, because the regex engine never has to backtrack.

### 2.6 Lookarounds

Sometimes you need to check what's around a match without including it in the result.

```python
# Look ahead: a number followed by "USD", capturing only the number
re.search(r"\d+(?=\s*USD)", "total 4500 USD").group(0)      # '4500'

# Look behind: a number preceded by a dollar sign
re.search(r"(?<=\$)\d+", "price $4500 net").group(0)         # '4500'

# Negative lookahead: "error" NOT followed by "_handled"
re.findall(r"error(?!_handled)", "error error_handled error")  # ['error', 'error']
```

| Syntax | Means |
|---|---|
| `(?=...)` | followed by |
| `(?!...)` | not followed by |
| `(?<=...)` | preceded by |
| `(?<!...)` | not preceded by |

Useful, but don't reach for them first. A lookaround in a pattern is often a sign the job would be better done with two simple steps than one clever one.

### 2.7 Patterns you'll actually use in a pipeline

**Parsing a log line:**

```python
line = '2026-08-22 14:03:11 ERROR [orders-etl] job_id=4412 rows=0 msg="no data found"'

pattern = re.compile(
    r"^(?P<ts>\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2})\s+"
    r"(?P<level>\w+)\s+"
    r"\[(?P<job>[\w-]+)\]"
)
pattern.search(line).groupdict()
# {'ts': '2026-08-22 14:03:11', 'level': 'ERROR', 'job': 'orders-etl'}
```

Notice the pattern is split across several strings. Python joins adjacent string literals automatically, so you can put each piece on its own line with a comment. Do this — long regexes on one line are unreadable and unreviewable.

**Extracting key=value pairs (and a real bug):**

```python
re.findall(r"(\w+)=(\S+)", line)
# [('job_id', '4412'), ('rows', '0'), ('msg', '"no')]
```

The last one is wrong. `\S+` means "non-whitespace," so it stopped at the space inside the quoted message. The fix is to say that a value is either a quoted string or a run of non-space characters:

```python
re.findall(r'(\w+)=("[^"]*"|\S+)', line)
# [('job_id', '4412'), ('rows', '0'), ('msg', '"no data found"')]
```

This is what regex debugging actually feels like. The pattern works on your first three test lines and quietly mangles the fourth. Test against ugly real data, not clean examples.

**Parsing an S3 / data lake path:**

```python
path = "s3://prod-lake/raw/orders/dt=2026-08-22/part-00017.parquet"

m = re.search(
    r"s3://(?P<bucket>[^/]+)/(?P<prefix>.+?)/dt=(?P<dt>\d{4}-\d{2}-\d{2})/(?P<file>[^/]+)$",
    path,
)
m.groupdict()
# {'bucket': 'prod-lake', 'prefix': 'raw/orders', 'dt': '2026-08-22', 'file': 'part-00017.parquet'}
```

Extracting the partition date from a path is something you'll do constantly.

**Cleaning column names for a warehouse:**

```python
def clean_column(name: str) -> str:
    name = re.sub(r"[^0-9a-zA-Z]+", "_", name)   # anything odd becomes underscore
    name = re.sub(r"_+", "_", name)              # collapse repeats
    return name.strip("_").lower()

clean_column(" Total Revenue ($CAD) ")   # 'total_revenue_cad'
clean_column("Customer ID#")             # 'customer_id'
```

Every messy CSV needs this. Worth keeping in a utils module.

**Validating before load:**

```python
PATTERNS = {
    "order_id":    re.compile(r"^ORD-\d{8}$"),
    "postal_code": re.compile(r"^[A-Z]\d[A-Z] ?\d[A-Z]\d$"),   # Canadian
    "email":       re.compile(r"^[^@\s]+@[^@\s]+\.[^@\s]+$"),
}

def validate(row: dict) -> list[str]:
    return [
        field for field, pattern in PATTERNS.items()
        if field in row and not pattern.match(str(row[field]))
    ]
```

On email specifically: do not go looking for the "complete" RFC-compliant email regex. It exists, it's thousands of characters long, and it's the wrong tool. Check for one `@` with something on each side and a dot after, then let the mail server be the real judge.

### 2.8 Two things that will hurt you

**Catastrophic backtracking.** Some patterns are exponentially slow on certain inputs. The classic shape is a repeat inside a repeat:

```python
re.match(r"^(a+)+$", "aaaaaaaaaaaaaaaaaaaaaaaaaaX")   # can hang for minutes
```

The engine tries every possible way to split those `a`s. Twenty-five characters, and it's effectively frozen. Avoid nesting quantifiers like `(x+)+` or `(x*)*`. If a pattern is ever slow, this is why.

**Regex is not a parser.** Don't parse JSON, XML, or HTML with regex. They're nested, and regex fundamentally can't handle nesting. Use `json.loads`, `lxml`, `BeautifulSoup`. Regex is for flat, line-shaped text, and it's excellent at that.

**Debugging tip:** paste your pattern into regex101.com, choose the Python flavour, and paste in real sample lines. It explains every token and highlights what matched. It will save you hours.

---

### Part 2 drills

Write these in `part2_regex/`.

**1.** Write patterns that match exactly these, anchored at both ends: a 4-digit year; a Canadian postal code (`K1A 0B1`, space optional); an order ID like `ORD-20260822-4412`; a Snowflake-style identifier (letters, digits, underscores, must not start with a digit).

**2.** From `"2026-08-22 14:03:11 ERROR [orders-etl] job_id=4412 rows=0"`, use named groups to produce a dict with `ts`, `level`, `job`, `job_id`, and `rows`.

**3.** Write `parse_log_line(line)` that returns a dict or `None`, then run it over a list of eight lines where three are malformed. Log the bad ones and return only the good ones.

**4.** Explain the difference in output between `r'msg="(.*)"'` and `r'msg="(.*?)"'` on a line containing two quoted messages. Then write a third version using a character class that's better than both.

**5.** Write `clean_column()` handling: leading/trailing spaces, mixed case, special characters, repeated underscores, and a name that starts with a digit (prefix it with `col_`).

**6.** From `"s3://lake/raw/orders/dt=2026-08-22/hour=14/part-0001.parquet"`, extract bucket, dataset, date, hour, and filename with named groups.

**7.** Use `re.sub` with a function as the replacement to mask emails: `"contact a.patel@corp.com now"` → `"contact a****@corp.com now"`. (Hint: `re.sub` accepts a callable that receives the match.)

**8.** Write `find_bad_rows(rows)` taking a list of dicts and returning `[(row_index, field_name), ...]` for every field failing its pattern. Include order_id, postal code, and a positive-decimal amount.

**9.** Using a negative lookahead, find every occurrence of `ERROR` in a log that is **not** followed by `_RECOVERED`.

**10.** Time `re.match(r"^(a+)+$", "a" * 20 + "X")` and then `re.match(r"^a+$", "a" * 20 + "X")`. Explain the gap in one sentence.

---

## PART 3 — PYTHON PATTERNS FOR DATA ENGINEERING

Concepts that show up constantly in pipeline code and rarely in beginner tutorials.

### 3.1 Generators — processing files bigger than memory

A generator produces values one at a time instead of building a list. `yield` instead of `return`.

```python
# Loads the entire file into memory. Dies on a 10 GB file.
def read_all(path):
    with open(path) as f:
        return [parse(line) for line in f]

# Holds one line at a time. Works on any size file.
def read_rows(path):
    with open(path) as f:
        for line in f:
            yield parse(line)
```

Same usage, completely different memory profile:

```python
for row in read_rows("huge.csv"):
    process(row)
```

Generators chain, and each stage still holds only one row:

```python
def read_rows(path): ...
def drop_nulls(rows):
    for r in rows:
        if r.get("id"):
            yield r
def enrich(rows):
    for r in rows:
        r["loaded_at"] = datetime.now(UTC)
        yield r

pipeline = enrich(drop_nulls(read_rows("huge.csv")))
for row in pipeline:
    write(row)
```

Nothing runs until you iterate. It's lazy all the way down. This is the same mental model as a Spark DAG, which is a useful thing to say out loud in an interview.

**The gotcha:** a generator is consumed once. Iterate it twice and the second loop sees nothing. If you need multiple passes, materialise it with `list()` and accept the memory cost, or read the source again.

### 3.2 Batching — because row-by-row writes are slow

Inserting one row at a time into a warehouse is death by network round-trip. Batch it.

Python 3.12 gives you this for free:

```python
from itertools import batched

for chunk in batched(read_rows("huge.csv"), 1000):
    warehouse.insert_many(chunk)      # chunk is a tuple of up to 1000 rows
```

Under 3.12, write it yourself — worth knowing anyway:

```python
from itertools import islice

def batched(iterable, n):
    it = iter(iterable)
    while chunk := tuple(islice(it, n)):
        yield chunk
```

Combines directly with async:

```python
async def load_all(rows, batch_size=1000, concurrency=5):
    sem = asyncio.Semaphore(concurrency)

    async def load_one(chunk):
        async with sem:
            await warehouse.insert_many(chunk)

    await asyncio.gather(*(load_one(c) for c in batched(rows, batch_size)))
```

That's generators for memory, batching for throughput, and async for concurrency, in nine lines. It's most of what an ingestion job is.

### 3.3 Context managers — guaranteed cleanup

`with` blocks guarantee cleanup runs, even on an exception. Write your own with `@contextmanager`:

```python
from contextlib import contextmanager
import time
import logging

logger = logging.getLogger(__name__)


@contextmanager
def timed(label: str):
    start = time.perf_counter()
    logger.info("%s starting", label)
    try:
        yield
    finally:
        logger.info("%s finished in %.2fs", label, time.perf_counter() - start)


with timed("extract orders"):
    rows = extract()
```

Everything before `yield` is setup, everything after is teardown, and `finally` means teardown happens even when the body raises.

A more useful one — transaction handling:

```python
@contextmanager
def transaction(conn):
    conn.execute("BEGIN")
    try:
        yield conn
        conn.execute("COMMIT")
    except Exception:
        conn.execute("ROLLBACK")
        raise
```

Async version uses `@asynccontextmanager` and `async with`.

### 3.4 Dataclasses — rows that aren't dicts

Dicts everywhere means typos are silent. `row["custommer_id"]` is a `KeyError` at runtime, or worse, a `.get()` that quietly returns `None`.

```python
from dataclasses import dataclass, field
from datetime import datetime, UTC


@dataclass(frozen=True, slots=True)
class OrderRow:
    order_id: str
    amount: float
    currency: str = "CAD"
    loaded_at: datetime = field(default_factory=lambda: datetime.now(UTC))

    def __post_init__(self):
        if self.amount < 0:
            raise ValueError(f"negative amount on {self.order_id}")
```

- `frozen=True` makes it immutable, so nothing downstream can silently mutate a row.
- `slots=True` cuts memory substantially, which matters at a million rows.
- `field(default_factory=...)` is required for mutable or computed defaults. Using `= []` or `= datetime.now()` directly creates **one shared object for every instance**, which is a genuinely nasty bug.
- `__post_init__` runs after construction — the natural place for validation.

For anything crossing a system boundary (an API response, a config file), use **Pydantic** instead. It coerces types and gives readable errors. Dataclasses are for internal structures, Pydantic for untrusted input.

### 3.5 Retry with exponential backoff

Networks fail. The correct response is to try again, but not immediately and not forever.

```python
import asyncio
import random
import logging

logger = logging.getLogger(__name__)


async def with_retry(fn, *args, attempts=5, base_delay=1.0, retry_on=(ConnectionError, TimeoutError)):
    for attempt in range(1, attempts + 1):
        try:
            return await fn(*args)
        except retry_on as exc:
            if attempt == attempts:
                logger.error("giving up after %d attempts: %s", attempts, exc)
                raise
            delay = base_delay * (2 ** (attempt - 1)) + random.uniform(0, 1)
            logger.warning("attempt %d failed (%s), retrying in %.1fs", attempt, exc, delay)
            await asyncio.sleep(delay)
```

Three details that matter:

**Exponential.** 1s, 2s, 4s, 8s. A struggling service needs room to recover; hammering it every 100ms makes the outage worse.

**Jitter.** The `random.uniform(0, 1)` is not decoration. Without it, 200 workers that failed simultaneously will all retry at exactly the same moment, producing a thundering herd that knocks the service over again. Jitter spreads them out.

**Only retry what's retryable.** A 503 is worth retrying. A 401 or a schema mismatch will fail identically five times while wasting fifteen seconds. Be specific in `retry_on`.

In production use the `tenacity` library rather than hand-rolling. Write it yourself once, then use the library.

### 3.6 Logging, not printing

`print` has no severity, no timestamp, no source, and can't be filtered or routed. Every pipeline you write will eventually run unattended, and then logs are all you have.

```python
import logging

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s %(levelname)s [%(name)s] %(message)s",
)
logger = logging.getLogger(__name__)

logger.info("loaded %d rows into %s", len(rows), table)      # ← note the commas
logger.warning("skipped %d malformed rows", bad_count)
logger.error("load failed for %s", table, exc_info=True)     # includes traceback
```

Two habits worth forming now:

**Use `%s` with commas, not f-strings.** `logger.info("loaded %d rows", n)` only formats the string if that level is actually enabled, and log aggregators can group by the message template. `logger.info(f"loaded {n} rows")` formats every time and produces a million unique message strings.

**`logger = logging.getLogger(__name__)` at the top of every module.** You get per-module filtering for free, so you can turn a noisy library down to WARNING without silencing your own code.

### 3.7 Caching lookups

```python
from functools import lru_cache


@lru_cache(maxsize=1000)
def get_currency_rate(code: str, date: str) -> float:
    return expensive_api_lookup(code, date)
```

Second call with the same arguments returns instantly. Ideal for dimension lookups repeated across many rows.

Two rules: **arguments must be hashable** (no lists or dicts — use tuples), and **don't cache anything that changes**. A cached "current" exchange rate that's four hours stale is a bug you'll debug for a long time. Cache lookups keyed by date, not lookups of "now."

---

### Part 3 drills

Write these in `part3_de_patterns/`.

**1.** Write a generator producing 1,000,000 fake rows. Sum one field with a `for` loop over the generator, then with `sum([...])` over a list comprehension. Compare memory using `tracemalloc`.

**2.** Chain three generators: read → filter out rows where `amount <= 0` → add a `loaded_at` field. Consume it and count. Confirm no intermediate list is ever built.

**3.** Implement `batched()` yourself with `islice`. Test it on 10 items with size 3 and confirm the last batch has 1.

**4.** Write a `@contextmanager` called `timed(label)` that logs elapsed time. Verify the timing still logs when the body raises an exception.

**5.** Write a `@contextmanager` simulating a database transaction with a fake connection object that records `["BEGIN", "COMMIT"]` or `["BEGIN", "ROLLBACK"]`. Test both paths.

**6.** Build an `OrderRow` dataclass with `frozen=True`, `slots=True`, a `default_factory` timestamp, and `__post_init__` validation rejecting negative amounts. Prove `frozen` works by trying to reassign a field.

**7.** Demonstrate the mutable default bug: build a dataclass with a list default using `field(default_factory=list)`, then explain in a comment what would go wrong with a bare `= []`.

**8.** Write `with_retry` with exponential backoff and jitter. Test with a function that fails twice then succeeds. Then test with one that always fails and confirm it gives up after the right number of attempts.

**9.** Configure logging with the format above. Log one line at each of INFO, WARNING, and ERROR (the last with `exc_info=True` inside an `except`). Then set the level to WARNING and confirm the INFO line disappears.

**10.** Add `@lru_cache` to a function that sleeps 0.1s. Call it 100 times with 5 distinct arguments. Time it, then print `func.cache_info()`.

**11.** Capstone. Build `pipeline.py` combining everything: a generator reading rows from a fake source, a regex validator marking bad rows, `batched()` for chunking, `asyncio` with a `Semaphore` to "load" chunks concurrently, retry with backoff on simulated failures, a `timed()` context manager around the whole thing, and a logged summary of rows read, rows rejected, batches loaded, and batches failed.

Drill 11 is the one to put on GitHub. It's a genuine ingestion job, and it's a better portfolio piece than most tutorials produce.

---

### A realistic schedule

You have nine days before the new role starts, and Snowflake needs some of them.

| Day | Focus |
|---|---|
| 1 | Setup + Part 1 sections 1.1–1.4, drills 1–3 |
| 2 | Part 1 sections 1.5–1.6, drills 4–5 |
| 3 | All of Part 2, drills 1–6 |
| 4 | Part 1 sections 1.7–1.9, drills 6–9 |
| 5 | Part 1 sections 1.10–1.11, drills 10–12 |
| 6 | Part 3 sections 3.1–3.4, drills 1–7 |
| 7 | Part 3 sections 3.5–3.7, drills 8–10 |
| 8 | Capstone (Part 3 drill 11) |
| 9 | Rest, or revisit whatever felt shakiest |

If you only get three days, do: Part 1 drill 2 (the ordering prediction), Part 1 drill 12 (the full ingestion job), and Part 3 drill 11 (the capstone). Those three cover most of what you'd actually be asked to write.

Post whatever breaks and I'll debug it with you.

---

# SECTION 4 — SOLUTIONS

Don't read these until you've tried. A solution you read is forgotten in a day; one you fought for sticks.

---

## Solutions — Section 1 (Async)

### 1.1 Sequential vs gather

```python
import asyncio
import time


async def fetch_table(name: str, seconds: float) -> dict:
    await asyncio.sleep(seconds)
    return {"table": name, "rows": 100}


async def main() -> None:
    tables = ["orders", "customers", "payments", "refunds", "invoices"]

    start = time.perf_counter()
    sequential = []
    for t in tables:
        sequential.append(await fetch_table(t, 1))
    print(f"sequential: {time.perf_counter() - start:.2f}s")

    start = time.perf_counter()
    concurrent = await asyncio.gather(*(fetch_table(t, 1) for t in tables))
    print(f"gather:     {time.perf_counter() - start:.2f}s")

    assert sequential == concurrent


asyncio.run(main())
```

Roughly 5.00s vs 1.00s. The `for` loop with `await` inside is fully sequential — each `await` finishes before the next iteration starts. That loop is the most common accidental way to write slow async code.

### 1.2 Output order

```
main start
a start
b start
a end
b end
main end
```

Why:

1. `main` prints, then hits `gather`, which schedules both coroutines.
2. `a` runs until its first `await`. Prints "a start", hits `sleep(0)`, gives up control.
3. `b` runs. Prints "b start", hits `sleep(0.1)`, gives up control.
4. `a`'s sleep was 0 seconds, so it's ready immediately. Prints "a end".
5. 0.1s later `b` wakes. Prints "b end".
6. Both done, `gather` returns, "main end".

The point: a coroutine runs **from its start to its first `await`** without interruption. Control only ever changes hands at an `await`. If you understand this, you understand the event loop.

`await asyncio.sleep(0)` is a useful idiom on its own — it means "let other tasks have a turn right now."

### 1.3 The blocking version

```python
import asyncio
import time


async def fetch_table_blocking(name: str, seconds: float) -> dict:
    time.sleep(seconds)          # blocking
    return {"table": name, "rows": 100}


async def main() -> None:
    start = time.perf_counter()
    await asyncio.gather(*(fetch_table_blocking(t, 1) for t in ["a", "b", "c", "d", "e"]))
    print(f"{time.perf_counter() - start:.2f}s")


asyncio.run(main())
```

About 5 seconds.

**One sentence:** `time.sleep` blocks the single thread the event loop runs on, so no other task can make progress while it sleeps, and the tasks run one after another exactly as if there were no async at all.

### 1.4 gather with return_exceptions

```python
import asyncio
import logging

logging.basicConfig(level=logging.INFO, format="%(levelname)s %(message)s")
logger = logging.getLogger(__name__)


async def load_file(name: str) -> str:
    await asyncio.sleep(0.2)
    if "corrupt" in name:
        raise ValueError(f"unreadable file: {name}")
    return f"{name} loaded"


async def main() -> None:
    files = ["a", "corrupt_b", "c", "corrupt_d", "e"]
    results = await asyncio.gather(
        *(load_file(f) for f in files),
        return_exceptions=True,
    )

    good, bad = [], []
    for name, result in zip(files, results):
        if isinstance(result, Exception):
            bad.append(name)
            logger.warning("failed: %s (%s)", name, result)
        else:
            good.append(name)

    print(f"loaded {len(good)} files, {len(bad)} failed")
    print(f"failures: {bad}")


asyncio.run(main())
```

`zip(files, results)` works because `gather` guarantees result order matches input order.

### 1.5 TaskGroup version

```python
import asyncio


async def load_file(name: str) -> str:
    await asyncio.sleep(0.2)
    if "corrupt" in name:
        raise ValueError(f"unreadable file: {name}")
    return f"{name} loaded"


async def main() -> None:
    files = ["a", "corrupt_b", "c", "corrupt_d", "e"]
    tasks = {}
    try:
        async with asyncio.TaskGroup() as tg:
            for f in files:
                tasks[f] = tg.create_task(load_file(f))
    except* ValueError as eg:
        print(f"{len(eg.exceptions)} files failed")
        for exc in eg.exceptions:
            print(f"  {exc}")

    for name, task in tasks.items():
        state = "cancelled" if task.cancelled() else ("failed" if task.exception() else "ok")
        print(f"{name}: {state}")


asyncio.run(main())
```

**What happens to the unstarted ones:** because all five sleep the same 0.2s, they all reach the failure point together and both corrupt files raise. If the delays differed, the first failure would cancel every sibling still in flight, and you'd see `cancelled` for tasks that had done nothing wrong.

That's the essential difference: `gather(return_exceptions=True)` lets everything finish; `TaskGroup` stops the whole batch at the first failure.

### 1.6 Retry with backoff

```python
import asyncio

call_count = 0


async def flaky_api() -> str:
    global call_count
    call_count += 1
    await asyncio.sleep(0.05)
    if call_count < 3:
        raise ConnectionError(f"failed on call {call_count}")
    return "success"


async def with_retry(fn, attempts: int = 5, base_delay: float = 1.0):
    for attempt in range(1, attempts + 1):
        try:
            return await fn()
        except ConnectionError as exc:
            if attempt == attempts:
                raise
            delay = base_delay * (2 ** (attempt - 1))
            print(f"attempt {attempt} failed ({exc}), retrying in {delay}s")
            await asyncio.sleep(delay)


print(asyncio.run(with_retry(flaky_api)))
```

Output: attempt 1 fails, waits 1s; attempt 2 fails, waits 2s; attempt 3 succeeds.

### 1.7 Jitter

```python
import random

delay = base_delay * (2 ** (attempt - 1)) + random.uniform(0, base_delay)
```

**Why it matters:** if 200 workers all hit the same outage at the same instant, without jitter all 200 retry at exactly t+1s, then all 200 at t+3s. Each retry wave is as large as the original load, so the struggling service gets hit by a synchronised spike right when it's trying to recover — a thundering herd. Jitter smears each wave across a window, turning a spike into a trickle.

### 1.8 Semaphore

```python
import asyncio

active = 0
peak = 0


async def fetch(sem: asyncio.Semaphore, i: int) -> int:
    global active, peak
    async with sem:
        active += 1
        peak = max(peak, active)
        await asyncio.sleep(0.1)
        active -= 1
        return i


async def main() -> None:
    sem = asyncio.Semaphore(5)
    await asyncio.gather(*(fetch(sem, i) for i in range(100)))
    print(f"peak concurrency: {peak}")


asyncio.run(main())
```

Prints exactly 5. Without the semaphore it would print 100.

Note this is only safe because there's no `await` between reading and writing `active` — a coroutine can't be interrupted except at an `await`, so those lines are effectively atomic. That's a genuinely useful property of async over threads.

### 1.9 Producer / consumer

```python
import asyncio


async def producer(queue: asyncio.Queue, n_records: int, n_workers: int) -> None:
    for i in range(n_records):
        await queue.put({"id": i})
        print(f"produced {i}")
    for _ in range(n_workers):
        await queue.put(None)


async def worker(queue: asyncio.Queue, name: str) -> None:
    while True:
        record = await queue.get()
        if record is None:
            queue.task_done()
            print(f"{name} stopping")
            break
        await asyncio.sleep(0.05)
        print(f"{name} handled record {record['id']}")
        queue.task_done()


async def main() -> None:
    n_workers = 3
    queue = asyncio.Queue(maxsize=100)
    await asyncio.gather(
        producer(queue, 20, n_workers),
        *(worker(queue, f"w{i}") for i in range(n_workers)),
    )


asyncio.run(main())
```

**With `maxsize=2`:** the producer prints "produced 0", "produced 1", then stops and waits. Production and consumption interleave instead of the producer racing ahead. That's backpressure — the queue can never hold more than 2 items, so memory stays bounded no matter how large the source is.

### 1.10 Async generator

```python
import asyncio


async def paginate(total_pages: int):
    for page in range(total_pages):
        await asyncio.sleep(0.05)
        yield [{"id": f"{page}-{i}"} for i in range(3)]


async def main() -> None:
    total = 0
    async for batch in paginate(10):
        total += len(batch)
        print(f"got {len(batch)} rows, running total {total}")
    print(f"final: {total}")


asyncio.run(main())
```

30 rows total. Only one page (3 rows) is ever in memory.

### 1.11 Timeout

```python
import asyncio


async def slow_call(seconds: float) -> str:
    await asyncio.sleep(seconds)
    return "real data"


async def fetch_with_timeout(seconds: float) -> str:
    try:
        async with asyncio.timeout(1):
            return await slow_call(seconds)
    except TimeoutError:
        print(f"timed out after 1s (call needed {seconds}s)")
        return "fallback"


async def main() -> None:
    print(await fetch_with_timeout(0.5))   # real data
    print(await fetch_with_timeout(2.0))   # fallback


asyncio.run(main())
```

Note that `asyncio.TimeoutError` became an alias for the builtin `TimeoutError` in Python 3.11, so either name works.

### 1.12 Capstone — full ingestion job

```python
import asyncio
import logging
import random

logging.basicConfig(level=logging.INFO, format="%(asctime)s %(levelname)s %(message)s")
logger = logging.getLogger("ingest")

MAX_CONCURRENT = 8
TIMEOUT_SECONDS = 2
MAX_ATTEMPTS = 3


async def fetch_table(name: str) -> dict:
    """Simulated source: sometimes slow, sometimes broken."""
    await asyncio.sleep(random.uniform(0.1, 0.5))
    roll = random.random()
    if roll < 0.15:
        raise ConnectionError(f"{name}: connection reset")
    if roll < 0.20:
        await asyncio.sleep(5)          # will hit the timeout
    return {"table": name, "rows": random.randint(100, 10_000)}


async def fetch_with_retry(name: str) -> dict:
    for attempt in range(1, MAX_ATTEMPTS + 1):
        try:
            async with asyncio.timeout(TIMEOUT_SECONDS):
                return await fetch_table(name)
        except (ConnectionError, TimeoutError) as exc:
            if attempt == MAX_ATTEMPTS:
                logger.error("%s: giving up after %d attempts (%s)", name, attempt, exc)
                raise
            delay = 0.2 * (2 ** (attempt - 1)) + random.uniform(0, 0.2)
            logger.warning("%s: attempt %d failed (%s), retry in %.2fs", name, attempt, exc, delay)
            await asyncio.sleep(delay)


async def fetch_guarded(sem: asyncio.Semaphore, name: str) -> dict:
    async with sem:
        return await fetch_with_retry(name)


async def main() -> None:
    tables = [f"table_{i:02d}" for i in range(50)]
    sem = asyncio.Semaphore(MAX_CONCURRENT)

    results = await asyncio.gather(
        *(fetch_guarded(sem, t) for t in tables),
        return_exceptions=True,
    )

    succeeded = [r for r in results if not isinstance(r, Exception)]
    failed = [(t, r) for t, r in zip(tables, results) if isinstance(r, Exception)]

    logger.info("=" * 50)
    logger.info("loaded  : %d tables", len(succeeded))
    logger.info("rows    : %s", f"{sum(r['rows'] for r in succeeded):,}")
    logger.info("failed  : %d tables", len(failed))
    for name, exc in failed:
        logger.info("   %s -> %s", name, type(exc).__name__)


asyncio.run(main())
```

The layering is the lesson. Semaphore wraps retry, retry wraps timeout, timeout wraps the call. Each layer does exactly one job, and you can reason about them independently. Retry inside the semaphore is deliberate — a task holds its slot across its retries instead of releasing and re-queueing, which keeps concurrency genuinely capped.

---

## Solutions — Section 2 (Regex)

### 2.1 Anchored patterns

```python
import re

YEAR        = re.compile(r"^\d{4}$")
POSTAL_CODE = re.compile(r"^[A-Z]\d[A-Z] ?\d[A-Z]\d$")
ORDER_ID    = re.compile(r"^ORD-\d{8}-\d{4}$")
IDENTIFIER  = re.compile(r"^[A-Za-z_]\w*$")

assert YEAR.match("2026") and not YEAR.match("202")
assert POSTAL_CODE.match("K1A 0B1") and POSTAL_CODE.match("K1A0B1")
assert ORDER_ID.match("ORD-20260822-4412")
assert IDENTIFIER.match("_orders_v2") and not IDENTIFIER.match("2orders")
```

`\w*` after the first character allows letters, digits, and underscores, but the leading `[A-Za-z_]` blocks a digit at the start. That mirrors how most SQL engines treat unquoted identifiers.

### 2.2 Named groups

```python
import re

LINE = re.compile(
    r"^(?P<ts>\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2})\s+"
    r"(?P<level>\w+)\s+"
    r"\[(?P<job>[\w-]+)\]\s+"
    r"job_id=(?P<job_id>\d+)\s+"
    r"rows=(?P<rows>\d+)"
)

line = "2026-08-22 14:03:11 ERROR [orders-etl] job_id=4412 rows=0"
print(LINE.search(line).groupdict())
# {'ts': '2026-08-22 14:03:11', 'level': 'ERROR', 'job': 'orders-etl',
#  'job_id': '4412', 'rows': '0'}
```

Everything comes back as a string. Cast in a separate step so a bad value produces a clear error:

```python
d = LINE.search(line).groupdict()
d["rows"] = int(d["rows"])
```

### 2.3 Parsing with rejection

```python
import logging
import re

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

LINE = re.compile(
    r"^(?P<ts>\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2})\s+"
    r"(?P<level>\w+)\s+"
    r"\[(?P<job>[\w-]+)\]"
)


def parse_log_line(line: str) -> dict | None:
    m = LINE.search(line)
    if m is None:
        return None
    return m.groupdict()


def parse_all(lines: list[str]) -> list[dict]:
    good, bad = [], 0
    for i, line in enumerate(lines):
        parsed = parse_log_line(line)
        if parsed is None:
            bad += 1
            logger.warning("line %d unparseable: %s", i, line[:60])
        else:
            good.append(parsed)
    logger.info("parsed %d lines, rejected %d", len(good), bad)
    return good


lines = [
    "2026-08-22 14:03:11 ERROR [orders-etl] job_id=1",
    "2026-08-22 14:03:12 INFO [orders-etl] job_id=2",
    "garbage line with no structure",
    "2026-08-22 14:03:13 WARN [payments-etl] job_id=3",
    "",
    "2026-08-22 14:03:14 INFO [orders-etl] job_id=4",
    "22/08/2026 14:03:15 INFO [orders-etl] job_id=5",   # wrong date format
    "2026-08-22 14:03:16 DEBUG [refunds-etl] job_id=6",
]
parse_all(lines)
```

Returning `None` rather than raising is the right call here. One bad line in a million shouldn't kill the job, but you do want it counted and logged. A silent skip is how you end up explaining missing data three weeks later.

### 2.4 Greedy, lazy, better

```python
import re

text = 'msg="first message" other stuff msg="second message"'

print(re.search(r'msg="(.*)"',  text).group(1))
# first message" other stuff msg="second message
print(re.search(r'msg="(.*?)"', text).group(1))
# first message
print(re.search(r'msg="([^"]*)"', text).group(1))
# first message
```

Greedy `.*` runs to the end of the line and backtracks until it can find a final quote, so it swallows both messages. Lazy `.*?` stops at the first quote it can.

The third is better than the lazy version because `[^"]*` *cannot* cross a quote at all. It expresses the intent directly, needs no backtracking, and won't surprise you when the pattern grows.

### 2.5 Column cleaner

```python
import re


def clean_column(name: str) -> str:
    name = name.strip()
    name = re.sub(r"[^0-9a-zA-Z]+", "_", name)
    name = re.sub(r"_+", "_", name)
    name = name.strip("_").lower()
    if re.match(r"^\d", name):
        name = f"col_{name}"
    return name


for raw in [" Total Revenue ($CAD) ", "Customer ID#", "2026 Sales",
            "order__id", "  ", "Amount (%)"]:
    print(f"{raw!r:28} -> {clean_column(raw)!r}")
```

```
' Total Revenue ($CAD) '     -> 'total_revenue_cad'
'Customer ID#'               -> 'customer_id'
'2026 Sales'                 -> 'col_2026_sales'
'order__id'                  -> 'order_id'
'  '                         -> ''
'Amount (%)'                 -> 'amount'
```

The empty-string case is worth handling explicitly in real code — raise, or fall back to `col_{position}`.

### 2.6 Path parsing

```python
import re

PATH = re.compile(
    r"s3://(?P<bucket>[^/]+)/"
    r"(?P<dataset>.+?)/"
    r"dt=(?P<date>\d{4}-\d{2}-\d{2})/"
    r"hour=(?P<hour>\d{1,2})/"
    r"(?P<filename>[^/]+)$"
)

path = "s3://lake/raw/orders/dt=2026-08-22/hour=14/part-0001.parquet"
print(PATH.search(path).groupdict())
# {'bucket': 'lake', 'dataset': 'raw/orders', 'date': '2026-08-22',
#  'hour': '14', 'filename': 'part-0001.parquet'}
```

`.+?` for the dataset is lazy on purpose. Greedy `.+` would swallow the `dt=` segment and then backtrack, and on deeply nested paths it can pick the wrong split.

### 2.7 Masking emails

```python
import re

EMAIL = re.compile(r"\b([\w.+-]+)@([\w-]+\.[\w.-]+)\b")


def mask(match: re.Match) -> str:
    local, domain = match.group(1), match.group(2)
    return f"{local[0]}{'*' * (len(local) - 1)}@{domain}"


print(EMAIL.sub(mask, "contact a.patel@corp.com or ops@corp.com now"))
# contact a******@corp.com or o**@corp.com now
```

Passing a function to `re.sub` is the trick worth remembering. The function receives the `Match` object and returns the replacement string, so the replacement can depend on what was matched. Useful for redaction, normalisation, and lookup-based substitution.

### 2.8 Row validator

```python
import re

PATTERNS = {
    "order_id":    re.compile(r"^ORD-\d{8}-\d{4}$"),
    "postal_code": re.compile(r"^[A-Z]\d[A-Z] ?\d[A-Z]\d$"),
    "amount":      re.compile(r"^\d+(\.\d{1,2})?$"),
}


def find_bad_rows(rows: list[dict]) -> list[tuple[int, str]]:
    problems = []
    for i, row in enumerate(rows):
        for field, pattern in PATTERNS.items():
            value = row.get(field)
            if value is None or not pattern.match(str(value)):
                problems.append((i, field))
    return problems


rows = [
    {"order_id": "ORD-20260822-4412", "postal_code": "K1A 0B1", "amount": "1250.00"},
    {"order_id": "ORD-123",           "postal_code": "K1A 0B1", "amount": "1250.00"},
    {"order_id": "ORD-20260822-4413", "postal_code": "90210",   "amount": "-50"},
    {"order_id": "ORD-20260822-4414", "postal_code": "M5V2T6",  "amount": "99.999"},
]
print(find_bad_rows(rows))
# [(1, 'order_id'), (2, 'postal_code'), (2, 'amount'), (3, 'amount')]
```

`^\d+(\.\d{1,2})?$` rejects negatives (no sign allowed) and rejects more than two decimal places. Returning `(index, field)` pairs rather than just "bad row" means the error message can name the actual problem, which is the difference between a useful quarantine table and a useless one.

### 2.9 Negative lookahead

```python
import re

log = "ERROR failed\nERROR_RECOVERED retry ok\nERROR timeout\nERROR_RECOVERED fine"
print(re.findall(r"ERROR(?!_RECOVERED)", log))
# ['ERROR', 'ERROR']
```

`(?!_RECOVERED)` checks what follows without consuming it, so the match itself is just `ERROR`. Without the lookahead you'd get four matches, because `ERROR` is a prefix of `ERROR_RECOVERED`.

### 2.10 Catastrophic backtracking

```python
import re
import time

for n in (18, 20, 22, 24):
    text = "a" * n + "X"

    start = time.perf_counter()
    re.match(r"^(a+)+$", text)
    nested = time.perf_counter() - start

    start = time.perf_counter()
    re.match(r"^a+$", text)
    simple = time.perf_counter() - start

    print(f"n={n}: nested {nested:8.4f}s   simple {simple:.6f}s")
```

**One sentence:** `(a+)+` gives the engine exponentially many ways to divide the same run of `a`s between the inner and outer repeat, so when the final `X` forces failure it has to try every one of them, while `a+` has only a single way to match and fails instantly.

Each extra character roughly doubles the time. Around n=25 you're waiting minutes. The rule that follows: never nest one quantifier inside another.

---

## Solutions — Section 3 (DE Patterns)

### 3.1 Generator vs list memory

```python
import tracemalloc


def gen_rows(n: int):
    for i in range(n):
        yield {"id": i, "amount": i * 1.5}


N = 1_000_000

tracemalloc.start()
total = sum(row["amount"] for row in gen_rows(N))
_, gen_peak = tracemalloc.get_traced_memory()
tracemalloc.stop()

tracemalloc.start()
rows = [{"id": i, "amount": i * 1.5} for i in range(N)]
total2 = sum(row["amount"] for row in rows)
_, list_peak = tracemalloc.get_traced_memory()
tracemalloc.stop()

print(f"generator peak: {gen_peak / 1024 / 1024:8.2f} MB")
print(f"list peak:      {list_peak / 1024 / 1024:8.2f} MB")
assert total == total2
```

The generator peak stays in kilobytes. The list peak runs to hundreds of megabytes. Same answer, and one of them scales to a file that doesn't fit in RAM.

### 3.2 Chained generators

```python
from datetime import datetime, UTC


def read_rows(n: int):
    for i in range(n):
        yield {"id": i, "amount": (i % 7) - 3}


def drop_non_positive(rows):
    for row in rows:
        if row["amount"] > 0:
            yield row


def add_loaded_at(rows):
    stamp = datetime.now(UTC)
    for row in rows:
        yield {**row, "loaded_at": stamp}


pipeline = add_loaded_at(drop_non_positive(read_rows(100)))
count = sum(1 for _ in pipeline)
print(f"{count} rows survived")
```

Nothing runs until `sum` starts iterating. Each row travels the full chain before the next is read, so memory holds one row regardless of source size.

### 3.3 Hand-rolled batched

```python
from itertools import islice


def batched(iterable, n: int):
    if n < 1:
        raise ValueError("n must be at least 1")
    it = iter(iterable)
    while chunk := tuple(islice(it, n)):
        yield chunk


for chunk in batched(range(10), 3):
    print(len(chunk), chunk)
# 3 (0, 1, 2)
# 3 (3, 4, 5)
# 3 (6, 7, 8)
# 1 (9,)
```

`iter(iterable)` at the top is essential. Without it, `islice` would restart from the beginning of a list every time. The walrus `:=` assigns and tests in one step; the loop ends when `islice` returns an empty tuple.

### 3.4 timed context manager

```python
import logging
import time
from contextlib import contextmanager

logging.basicConfig(level=logging.INFO, format="%(levelname)s %(message)s")
logger = logging.getLogger(__name__)


@contextmanager
def timed(label: str):
    start = time.perf_counter()
    logger.info("%s starting", label)
    try:
        yield
    finally:
        logger.info("%s finished in %.3fs", label, time.perf_counter() - start)


with timed("clean run"):
    time.sleep(0.1)

try:
    with timed("failing run"):
        time.sleep(0.05)
        raise ValueError("boom")
except ValueError:
    print("caught it, but the timing still logged")
```

The `finally` is what makes it reliable. Without it, an exception would skip the teardown and you'd lose timing data on exactly the runs you most want to investigate.

### 3.5 Transaction context manager

```python
from contextlib import contextmanager


class FakeConnection:
    def __init__(self):
        self.log = []

    def execute(self, sql: str):
        self.log.append(sql)


@contextmanager
def transaction(conn: FakeConnection):
    conn.execute("BEGIN")
    try:
        yield conn
        conn.execute("COMMIT")
    except Exception:
        conn.execute("ROLLBACK")
        raise


conn = FakeConnection()
with transaction(conn) as c:
    c.execute("INSERT INTO orders VALUES (1)")
print(conn.log)   # ['BEGIN', 'INSERT INTO orders VALUES (1)', 'COMMIT']

conn2 = FakeConnection()
try:
    with transaction(conn2) as c:
        c.execute("INSERT INTO orders VALUES (2)")
        raise ValueError("bad row")
except ValueError:
    pass
print(conn2.log)  # ['BEGIN', 'INSERT INTO orders VALUES (2)', 'ROLLBACK']
```

The bare `raise` after ROLLBACK matters. Roll back, then let the caller find out something went wrong. Swallowing the exception here would report success on a transaction that didn't commit.

### 3.6 OrderRow

```python
from dataclasses import dataclass, field, FrozenInstanceError
from datetime import datetime, UTC


@dataclass(frozen=True, slots=True)
class OrderRow:
    order_id: str
    amount: float
    currency: str = "CAD"
    loaded_at: datetime = field(default_factory=lambda: datetime.now(UTC))

    def __post_init__(self):
        if self.amount < 0:
            raise ValueError(f"negative amount on {self.order_id}: {self.amount}")
        if not self.order_id:
            raise ValueError("order_id is required")


row = OrderRow("ORD-1", 1250.00)
print(row)

try:
    row.amount = 99
except FrozenInstanceError:
    print("frozen: cannot reassign")

try:
    OrderRow("ORD-2", -5)
except ValueError as e:
    print(f"validation: {e}")
```

`__post_init__` works with `frozen=True` for validation because it only *reads* the fields. If you needed to *modify* one inside `__post_init__` on a frozen dataclass, you'd have to use `object.__setattr__(self, "field", value)`, which is a strong hint your design wants a classmethod constructor instead.

### 3.7 Mutable default

```python
from dataclasses import dataclass, field


@dataclass
class Batch:
    name: str
    errors: list[str] = field(default_factory=list)   # correct

    # errors: list[str] = []
    # ^ Python raises ValueError at class definition time for list/dict/set.
    #
    # The real danger is the version it does NOT catch:
    #     loaded_at: datetime = datetime.now()
    # That is evaluated ONCE, when the class is defined. Every instance you
    # ever create gets the timestamp of program start, not of its own creation.
    # A pipeline running for six hours would stamp every row with the same
    # time, and nothing would error. default_factory defers evaluation to
    # instance creation, which is what you actually want.


a, b = Batch("a"), Batch("b")
a.errors.append("bad row 1")
print(a.errors, b.errors)   # ['bad row 1'] []  ← separate lists, correct
```

### 3.8 with_retry

```python
import asyncio
import logging
import random

logging.basicConfig(level=logging.INFO, format="%(levelname)s %(message)s")
logger = logging.getLogger(__name__)


async def with_retry(fn, *args, attempts=5, base_delay=0.1,
                     retry_on=(ConnectionError, TimeoutError)):
    for attempt in range(1, attempts + 1):
        try:
            return await fn(*args)
        except retry_on as exc:
            if attempt == attempts:
                logger.error("gave up after %d attempts: %s", attempts, exc)
                raise
            delay = base_delay * (2 ** (attempt - 1)) + random.uniform(0, base_delay)
            logger.warning("attempt %d failed (%s), retry in %.3fs", attempt, exc, delay)
            await asyncio.sleep(delay)


def make_flaky(fail_times: int):
    state = {"calls": 0}

    async def flaky():
        state["calls"] += 1
        if state["calls"] <= fail_times:
            raise ConnectionError(f"failure {state['calls']}")
        return f"ok after {state['calls']} calls"

    return flaky, state


async def main():
    fn, state = make_flaky(2)
    print(await with_retry(fn))

    fn2, state2 = make_flaky(99)
    try:
        await with_retry(fn2, attempts=4)
    except ConnectionError:
        print(f"gave up correctly after {state2['calls']} calls")


asyncio.run(main())
```

`make_flaky` returns a closure over a mutable dict instead of using `global`. Cleaner, and it lets you build several independent flaky functions in one test.

### 3.9 Logging levels

```python
import logging

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s %(levelname)s [%(name)s] %(message)s",
)
logger = logging.getLogger(__name__)

logger.info("loaded %d rows into %s", 12345, "orders")
logger.warning("skipped %d malformed rows", 7)

try:
    1 / 0
except ZeroDivisionError:
    logger.error("load failed for %s", "orders", exc_info=True)

logging.getLogger().setLevel(logging.WARNING)
logger.info("this will not appear")
logger.warning("this will")
```

`exc_info=True` attaches the full traceback. `logger.exception(...)` is shorthand for the same thing inside an `except` block.

### 3.10 lru_cache

```python
import time
from functools import lru_cache


@lru_cache(maxsize=None)
def get_rate(currency: str) -> float:
    time.sleep(0.1)
    return {"CAD": 1.0, "USD": 1.35, "EUR": 1.47, "GBP": 1.71, "JPY": 0.009}[currency]


start = time.perf_counter()
for i in range(100):
    get_rate(["CAD", "USD", "EUR", "GBP", "JPY"][i % 5])
print(f"{time.perf_counter() - start:.3f}s")
print(get_rate.cache_info())
```

About 0.5s instead of 10s — five real calls, ninety-five cache hits. `cache_info()` shows `hits=95, misses=5`, and that hit ratio is the number to watch. A low ratio means your key has too much variety for caching to help.

### 3.11 Capstone pipeline

```python
"""pipeline.py — a small but complete ingestion job."""

import asyncio
import logging
import random
import re
import time
from contextlib import contextmanager
from dataclasses import dataclass, field
from datetime import datetime, UTC
from itertools import islice

logging.basicConfig(level=logging.INFO, format="%(asctime)s %(levelname)s %(message)s")
logger = logging.getLogger("pipeline")

BATCH_SIZE = 100
MAX_CONCURRENT = 5
MAX_ATTEMPTS = 3

ORDER_ID = re.compile(r"^ORD-\d{8}-\d{4}$")
AMOUNT = re.compile(r"^\d+(\.\d{1,2})?$")


# ---------- stats ----------

@dataclass
class Stats:
    read: int = 0
    rejected: int = 0
    batches_loaded: int = 0
    batches_failed: int = 0
    reasons: dict = field(default_factory=dict)

    def reject(self, reason: str) -> None:
        self.rejected += 1
        self.reasons[reason] = self.reasons.get(reason, 0) + 1


# ---------- utilities ----------

@contextmanager
def timed(label: str):
    start = time.perf_counter()
    logger.info("%s starting", label)
    try:
        yield
    finally:
        logger.info("%s finished in %.2fs", label, time.perf_counter() - start)


def batched(iterable, n: int):
    it = iter(iterable)
    while chunk := tuple(islice(it, n)):
        yield chunk


# ---------- extract ----------

def read_source(n: int, stats: Stats):
    """Generator: one row at a time, some deliberately malformed."""
    for i in range(n):
        stats.read += 1
        roll = random.random()
        if roll < 0.05:
            yield {"order_id": f"BAD-{i}", "amount": "100.00"}
        elif roll < 0.10:
            yield {"order_id": f"ORD-20260822-{i:04d}", "amount": "-99"}
        else:
            yield {"order_id": f"ORD-20260822-{i:04d}", "amount": f"{random.uniform(10, 999):.2f}"}


# ---------- validate ----------

def validate(rows, stats: Stats):
    for row in rows:
        if not ORDER_ID.match(row["order_id"]):
            stats.reject("bad_order_id")
            continue
        if not AMOUNT.match(row["amount"]):
            stats.reject("bad_amount")
            continue
        yield {**row, "amount": float(row["amount"]), "loaded_at": datetime.now(UTC)}


# ---------- load ----------

async def write_batch(chunk: tuple) -> int:
    await asyncio.sleep(random.uniform(0.05, 0.2))
    if random.random() < 0.20:
        raise ConnectionError("warehouse connection reset")
    return len(chunk)


async def write_with_retry(chunk: tuple) -> int:
    for attempt in range(1, MAX_ATTEMPTS + 1):
        try:
            return await write_batch(chunk)
        except ConnectionError as exc:
            if attempt == MAX_ATTEMPTS:
                raise
            delay = 0.1 * (2 ** (attempt - 1)) + random.uniform(0, 0.1)
            logger.warning("batch retry %d after %s, waiting %.2fs", attempt, exc, delay)
            await asyncio.sleep(delay)


async def load_all(batches, stats: Stats) -> None:
    sem = asyncio.Semaphore(MAX_CONCURRENT)

    async def guarded(chunk):
        async with sem:
            return await write_with_retry(chunk)

    results = await asyncio.gather(
        *(guarded(c) for c in batches),
        return_exceptions=True,
    )
    for r in results:
        if isinstance(r, Exception):
            stats.batches_failed += 1
            logger.error("batch permanently failed: %s", r)
        else:
            stats.batches_loaded += 1


# ---------- orchestrate ----------

async def main() -> None:
    stats = Stats()
    with timed("orders ingestion"):
        rows = read_source(5_000, stats)
        clean = validate(rows, stats)
        batches = list(batched(clean, BATCH_SIZE))   # materialised: generators aren't concurrent-safe
        await load_all(batches, stats)

    logger.info("=" * 46)
    logger.info("rows read      : %s", f"{stats.read:,}")
    logger.info("rows rejected  : %s %s", f"{stats.rejected:,}", stats.reasons)
    logger.info("batches loaded : %d", stats.batches_loaded)
    logger.info("batches failed : %d", stats.batches_failed)


if __name__ == "__main__":
    asyncio.run(main())
```

Two design notes worth understanding, because they're the parts people get wrong:

**Why `list(batched(...))`.** The read-validate-batch chain is a lazy generator, which is exactly what you want for memory. But `asyncio.gather` needs all the coroutines up front, and a plain generator can't be safely pulled from by several concurrent tasks. Materialising the list of batches is the pragmatic fix at this scale. For a genuinely huge source you'd instead feed the generator into an `asyncio.Queue` with a `maxsize` and let a fixed pool of workers pull from it — that's the pattern from Part 1 section 1.9, and it keeps memory bounded end to end.

**Why rejected rows are counted by reason.** "3,000 rows rejected" tells you something is wrong. "2,940 bad_amount, 60 bad_order_id" tells you an upstream currency column changed format. The dict costs one line and saves an afternoon.

---

### What to do when something breaks

1. Read the last line of the traceback first. It names the actual error.
2. `<coroutine object ...>` in your output means a missing `await`.
3. Async code taking exactly as long as sync code means a blocking call.
4. `AttributeError: 'NoneType' object has no attribute 'group'` means your regex didn't match — print the input line and check it in regex101.
5. If none of that lands, paste the code and the traceback and I'll work through it with you.
