# Data Modelling, Explained Like You're Five

### One-file guide — setup, lessons, drills, and solutions

You know SQL. You've built pipelines. This is about the part nobody teaches: **how to decide what tables should exist and what each row should mean.**

Everything is explained through one toy shop. Same shop, all the way through. Every query in here was run before it was written, so the outputs are real.

---

## Contents

- **[Section 0 — Setup](#section-0--setup)** · DuckDB in your Codespace, 3 minutes
- **[Section 1 — What modelling even is](#section-1--what-modelling-even-is)**
- **[Section 2 — Grain: the one question that matters](#section-2--grain-the-one-question-that-matters)**
- **[Section 3 — Facts and dimensions](#section-3--facts-and-dimensions)**
- **[Section 4 — Keys](#section-4--keys)**
- **[Section 5 — The star schema](#section-5--the-star-schema)**
- **[Section 6 — Why not one big table?](#section-6--why-not-one-big-table)**
- **[Section 7 — When things change: SCDs](#section-7--when-things-change-scds)**
- **[Section 8 — The fan-out trap](#section-8--the-fan-out-trap)**
- **[Section 9 — Layers](#section-9--layers)**
- **[Section 10 — Snowflake specifics](#section-10--snowflake-specifics)**
- **[Section 11 — The recipe](#section-11--the-recipe)**
- **[Section 12 — Mistakes](#section-12--mistakes)**
- **[Drills](#drills)** · 14 of them
- **[Solutions](#solutions)**

**How to use this:** read a section, then run its SQL yourself. Type it, don't paste. Do the drills for that section before moving on.

---

# SECTION 0 — SETUP

You need somewhere to run SQL. Snowflake works, but it's slow to iterate on and you'll burn trial credits. **DuckDB** is better for learning: it's a single file, no server, no account, and the SQL is close enough to Snowflake that everything transfers.

In your Codespace terminal:

```bash
mkdir -p ~/modelling && cd ~/modelling
pip install duckdb --break-system-packages
```

Create `seed.sql`:

```sql
CREATE OR REPLACE TABLE raw_sales (
  receipt_no   VARCHAR,
  sold_at      TIMESTAMP,
  kid_name     VARCHAR,
  kid_city     VARCHAR,
  toy_name     VARCHAR,
  toy_category VARCHAR,
  toy_price    DECIMAL(10,2),
  quantity     INTEGER,
  shop_name    VARCHAR
);

INSERT INTO raw_sales VALUES
 ('R-001','2026-08-01 10:15','Ana','Brampton','Red Robot','Robots',24.99,2,'Main Street'),
 ('R-001','2026-08-01 10:15','Ana','Brampton','Puzzle Cube','Puzzles',9.50,1,'Main Street'),
 ('R-002','2026-08-01 11:40','Ben','Toronto','Red Robot','Robots',24.99,1,'Main Street'),
 ('R-003','2026-08-02 09:05','Ana','Brampton','Kite','Outdoor',14.00,3,'Lakeside'),
 ('R-004','2026-08-02 16:20','Cara','Toronto','Puzzle Cube','Puzzles',9.50,4,'Lakeside'),
 ('R-005','2026-08-03 12:00','Ben','Mississauga','Red Robot','Robots',29.99,1,'Main Street');
```

Load it and open a shell:

```bash
python3 -c "import duckdb; duckdb.connect('shop.duckdb').execute(open('seed.sql').read())"
python3 -c "import duckdb; duckdb.connect('shop.duckdb').sql('SELECT * FROM raw_sales').show()"
```

For interactive work, `pip install duckdb-cli` isn't needed — just run `python3` and:

```python
import duckdb
con = duckdb.connect("shop.duckdb")
con.sql("SELECT * FROM raw_sales").show()
```

Six rows. That's your whole world for this guide. Small on purpose — you can check every answer by counting on your fingers, which is exactly how you catch modelling mistakes.

---

# SECTION 1 — WHAT MODELLING EVEN IS

## The toy shop

You run a toy shop. Every time someone buys something, you scribble a line in a notebook:

> *Ana bought 2 Red Robots and 1 Puzzle Cube at the Main Street shop on Saturday morning. Robots are $24.99, puzzles are $9.50.*

That's `raw_sales`. It works fine for one day. But after a year the notebook has a hundred thousand lines, and some problems show up:

**You wrote "Red Robot" a thousand times.** If you decide to rename it "Robot (Red)", you now have to fix a thousand lines. Miss three, and your report says you sold two different toys.

**You can't answer simple questions.** "How many kids live in Toronto?" You have to read every line and try to remember which names you've already counted. Ben appears twice with two different cities — did he move, or did someone type it wrong?

**Nobody knows what a line means.** Is one line one *sale*, or one *toy*? Ana's purchase took two lines. If someone counts lines and says "we had 6 sales", they're wrong — there were 5.

**Data modelling is deciding how to split that one messy notebook into a few tidy books so those problems go away.**

That's the whole thing. Everything else is technique.

## The two tidy books

You'd naturally split it like this:

**A book of things that happened.** One line per toy sold. Short lines, just numbers and codes. This book gets huge, because things keep happening.

**Small books of things that exist.** One book listing every toy, one listing every kid, one listing every shop. Each thing appears exactly once. These stay small.

Grown-up names:

- The book of things that happened is a **fact table**
- The small books of things that exist are **dimension tables**

That's it. That's the core of dimensional modelling, and it covers most of what a data engineer does.

---

# SECTION 2 — GRAIN: THE ONE QUESTION THAT MATTERS

Before you write a single `CREATE TABLE`, answer this:

> **What does one row mean?**

That sentence is called the **grain**. Getting it wrong is the single most expensive mistake in data engineering, and getting it right prevents about half of all data bugs.

Look at `raw_sales`. Ana's receipt R-001 takes up two rows, because she bought two different toys. So:

- **Wrong:** "one row = one sale"
- **Right:** "one row = one toy on one receipt"

Say it out loud as a full sentence. If you can't, you don't have a grain yet.

Some real examples:

| Table | Grain |
|---|---|
| `fct_sale_line` | one product on one receipt |
| `fct_order` | one order |
| `fct_daily_balance` | one account on one day |
| `dim_kid` | one kid |
| `dim_date` | one calendar day |

**How to check your grain in SQL.** Count rows, then count distinct combinations of the columns you *think* define a row. They must match.

```sql
SELECT count(*) AS n_rows,
       count(DISTINCT (receipt_no, toy_name)) AS n_unique
FROM raw_sales;
-- 6, 6  ✓ grain holds
```

If `n_rows` is bigger, you have duplicates, or your grain is finer than you thought. Run this check on every fact table you build. Put it in a test. This is genuinely the highest-value five lines of SQL in your job.

**Never mix grains in one table.** If you put "one toy on one receipt" rows *and* "one whole receipt" rows in the same table, every sum you ever write will be wrong, and it will take someone three weeks to notice.

---

# SECTION 3 — FACTS AND DIMENSIONS

## The kid test

How do you know if a column belongs in the fact table or a dimension?

Ask: **does it describe something that happened, or something that exists?**

| Column | Happened or exists? | Goes in |
|---|---|---|
| quantity | happened (2 were sold) | fact |
| line total | happened | fact |
| toy category | exists (robots are always robots) | dimension |
| kid's city | exists | dimension |
| shop name | exists | dimension |
| sold at | happened — but it's *when* | it's the link to the date dimension |

Simpler still:

- **Numbers you'd want to add up** → fact
- **Words you'd want to group by or filter on** → dimension

"Total revenue **by** category **for** Toronto kids **in** August." The thing after "total" is a fact. Everything after "by", "for", and "in" is a dimension.

## Facts are skinny, dimensions are wide

A fact table row is mostly just ID numbers and a few measures:

```
date_key | kid_key | toy_key | shop_key | quantity | unit_price | line_total
20260801 |       1 |       3 |       2 |        2 |      24.99 |      49.98
```

Boring on purpose. Billions of these rows exist, so every extra byte costs real money.

A dimension row is chatty:

```
toy_key | toy_name  | toy_category | brand   | colour | age_min | is_battery_powered
      3 | Red Robot | Robots       | ToyCo   | red    |       6 | true
```

Only a few thousand of these, so extra columns are cheap. **Put descriptive columns in dimensions and be generous about it.** Every column you add there is a new way someone can slice the data without asking you for help.

## Three kinds of measure

Not all numbers add up the same way, and this catches people out.

**Additive** — adds up across everything. `quantity`, `line_total`. Sum them by day, by kid, by toy, any combination. Fine.

**Semi-additive** — adds up across some things but not time. An account balance: summing Monday's and Tuesday's balance is meaningless. You'd take the *last* value, not the sum.

**Non-additive** — never add these. Ratios, percentages, unit prices. Summing `unit_price` gives you a number that means nothing. Store the *ingredients* (total revenue, total units) and compute the ratio at the end.

Rule: **store the parts, divide at the end.** Never store the average.

---

# SECTION 4 — KEYS

## Natural keys are sticky nametags

Right now you link things by name: `toy_name = 'Red Robot'`. That's a **natural key** — it comes from the real world.

Problem: real-world things change. The toy gets renamed. A kid gets married and changes surname. A shop moves. Suddenly your joins break, or worse, silently match the wrong things.

Also, "Red Robot" is 9 bytes repeated a billion times in your fact table. `3` is 4 bytes.

## Surrogate keys are numbered stickers

So you give every toy a sticker with a number on it. The number means nothing. It never changes. That's a **surrogate key**.

```sql
CREATE OR REPLACE TABLE dim_toy AS
SELECT
  row_number() OVER (ORDER BY toy_name) AS toy_key,   -- the sticker
  toy_name,                                            -- the natural key
  toy_category
FROM (SELECT DISTINCT toy_name, toy_category FROM raw_sales);
```

```
toy_key | toy_name    | toy_category
      1 | Kite        | Outdoor
      2 | Puzzle Cube | Puzzles
      3 | Red Robot   | Robots
```

Keep the natural key as a normal column. You'll need it to match against source data, and it's the first thing you check when a join looks wrong.

**One exception worth knowing.** Date dimensions usually use a readable integer key: `20260801` for 1 August 2026. Everyone does this. It makes debugging much easier, because you can read a date key at a glance, and it sorts correctly.

## The date dimension

Almost every model has one. It's a table with one row per calendar day and a column for every way anyone might want to slice time.

```sql
CREATE OR REPLACE TABLE dim_date AS
SELECT
  CAST(strftime(d, '%Y%m%d') AS INTEGER) AS date_key,
  d                                      AS full_date,
  dayname(d)                             AS day_name,
  CAST(strftime(d,'%m') AS INTEGER)      AS month_no,
  dayofweek(d) IN (0,6)                  AS is_weekend
FROM (SELECT DISTINCT CAST(sold_at AS DATE) AS d FROM raw_sales);
```

```
date_key | full_date  | day_name | month_no | is_weekend
20260801 | 2026-08-01 | Saturday |        8 | true
20260802 | 2026-08-02 | Sunday   |        8 | true
20260803 | 2026-08-03 | Monday   |        8 | false
```

**Why bother, when SQL has date functions?** Because "is this a public holiday in Ontario?" and "which fiscal quarter is this?" are not SQL functions. They're business facts. A date dimension is where they live, defined once, so every report agrees.

In real life you generate 30 years of dates up front, not just the days you happen to have sales for.

---

# SECTION 5 — THE STAR SCHEMA

Put the fact table in the middle. Put the dimensions around it. Draw the joins. It looks like a star.

```
              dim_date
                  |
   dim_kid --- fct_sale_line --- dim_toy
                  |
              dim_shop
```

Here's the whole thing built:

```sql
CREATE OR REPLACE TABLE dim_kid AS
SELECT row_number() OVER (ORDER BY kid_name) AS kid_key, kid_name
FROM (SELECT DISTINCT kid_name FROM raw_sales);

CREATE OR REPLACE TABLE dim_shop AS
SELECT row_number() OVER (ORDER BY shop_name) AS shop_key, shop_name
FROM (SELECT DISTINCT shop_name FROM raw_sales);

CREATE OR REPLACE TABLE fct_sale_line AS
SELECT
  s.receipt_no,                                          -- degenerate dimension
  CAST(strftime(s.sold_at,'%Y%m%d') AS INTEGER) AS date_key,
  k.kid_key,
  t.toy_key,
  sh.shop_key,
  s.quantity,
  s.toy_price               AS unit_price,
  s.quantity * s.toy_price  AS line_total
FROM raw_sales s
JOIN dim_kid  k  ON k.kid_name   = s.kid_name
JOIN dim_toy  t  ON t.toy_name   = s.toy_name
JOIN dim_shop sh ON sh.shop_name = s.shop_name;
```

Now questions become easy and, more importantly, *the same shape every time*:

```sql
SELECT t.toy_category, sum(f.line_total) AS revenue, sum(f.quantity) AS units
FROM fct_sale_line f
JOIN dim_toy t USING (toy_key)
GROUP BY 1
ORDER BY revenue DESC;
```

```
toy_category | revenue | units
Robots       |  104.96 |     4
Puzzles      |   47.50 |     5
Outdoor      |   42.00 |     3
```

Every analytical question is now: pick the fact table, join the dimensions you need, group by dimension columns, sum fact columns. That predictability is the actual product. Analysts stop asking you how to join things.

## Degenerate dimensions

`receipt_no` sits in the fact table with no dimension of its own. That's fine and it has a name: a **degenerate dimension**. It's an identifier you want to group by ("how many toys per receipt?") but there's nothing else to say about a receipt. Don't build `dim_receipt` with a single column in it.

## Star vs snowflake

If you split `dim_toy` further — a separate `dim_category` table that `dim_toy` points to — the diagram grows branches and it's called a **snowflake schema**. (No relation to the warehouse; the name is much older.)

Mostly, don't. It saves a trivial amount of space and costs an extra join on every single query. Keep dimensions flat and wide. The rare exception is a dimension shared by many others and genuinely large.

---

# SECTION 6 — WHY NOT ONE BIG TABLE?

Fair question, and it comes up in every design review.

## The case for one big table

Modern warehouses are columnar. They only read the columns you ask for, so a 200-column table isn't slow if you select 3 columns. Joins cost something. Storage is cheap. So why not just keep `raw_sales` and query it?

For a single dashboard with one team, honestly — you can. Don't build a star schema for one report.

## Where it falls apart

**Renaming.** Change "Robots" to "Robotics" and you rewrite a billion fact rows. In a star, you update one row in `dim_toy`.

**Things that don't appear.** "Which toys sold nothing in August?" One big table can't answer this — toys with no sales have no rows. `dim_toy` has every toy, so a left join finds them instantly. This class of question is invisible until someone asks it, and then it's a rebuild.

**History.** When the Red Robot's price changed from $24.99 to $29.99, one big table just… has both numbers, with no way to know which was correct when. Section 7 is entirely about this.

**Different grains.** Sales happen per line. Inventory counts happen per day. Returns happen per item. These are three fact tables sharing the same dimensions. You cannot flatten them into one table without inventing rows that don't mean anything.

**Agreement.** With one big table per team, marketing's "revenue" and finance's "revenue" will differ, and every meeting starts by arguing about whose number is right. Shared dimensions are how organisations agree.

## The actual answer

Use a star schema for the **core** layer that many people build on. Build **one big flat table** on top of it for a specific dashboard that needs to be fast and simple. Both, in that order. Section 9 shows where each one lives.

---

# SECTION 7 — WHEN THINGS CHANGE: SCDs

The Red Robot cost $24.99 in July and $29.99 in August. Ben lived in Toronto, then moved to Mississauga.

What should `dim_kid` say about Ben?

This is the **slowly changing dimension** problem — "slowly" because kids move house occasionally, not constantly. There are two answers you actually need.

## Type 1 — overwrite

Just change it. Ben's row now says Mississauga. His old orders now look like they came from Mississauga, even though they didn't.

```sql
UPDATE dim_kid SET kid_city = 'Mississauga' WHERE kid_name = 'Ben';
```

Simple. Use it when history genuinely doesn't matter, or when the old value was just a typo.

## Type 2 — keep both, with dates

Add a new row. Keep the old one. Mark the time window each was true for.

Think of it like this: your friend moves house. You don't erase the birthday card you sent them last year — it really did go to the old address. You just start using the new one from now on.

```sql
CREATE OR REPLACE TABLE dim_toy_scd2 (
  toy_key INTEGER, toy_name VARCHAR, toy_category VARCHAR, list_price DECIMAL(10,2),
  valid_from DATE, valid_to DATE, is_current BOOLEAN
);

INSERT INTO dim_toy_scd2 VALUES
 (1,'Kite','Outdoor',14.00, DATE '2026-01-01', DATE '9999-12-31', TRUE),
 (2,'Puzzle Cube','Puzzles',9.50, DATE '2026-01-01', DATE '9999-12-31', TRUE),
 (3,'Red Robot','Robots',24.99, DATE '2026-01-01', DATE '2026-08-02', FALSE),
 (4,'Red Robot','Robots',29.99, DATE '2026-08-03', DATE '9999-12-31', TRUE);
```

Notice: **the Red Robot now has two rows and two different surrogate keys.** `toy_key` 3 is the old version, 4 is the new one. `toy_name` stays the same in both — that's the natural key, and it's how you know they're the same toy.

Three columns do the work:

- `valid_from` / `valid_to` — the window this version was true
- `is_current` — a shortcut flag so the common query is easy
- `9999-12-31` as "still true" — an actual date, so `BETWEEN` works without special-casing NULLs

### Asking "what was true then?"

```sql
SELECT s.receipt_no, CAST(s.sold_at AS DATE) AS d, t.list_price AS price_then
FROM raw_sales s
JOIN dim_toy_scd2 t
  ON t.toy_name = s.toy_name
 AND CAST(s.sold_at AS DATE) BETWEEN t.valid_from AND t.valid_to
WHERE s.toy_name = 'Red Robot';
```

```
receipt_no | d          | price_then
R-001      | 2026-08-01 |      24.99
R-002      | 2026-08-01 |      24.99
R-005      | 2026-08-03 |      29.99
```

That's the payoff. August 1st sales are priced correctly at the old price, August 3rd at the new one. A Type 1 dimension would have said $29.99 for all three, and your July revenue report would have been wrong.

### Asking "what's true now?"

```sql
SELECT toy_name, list_price FROM dim_toy_scd2 WHERE is_current;
```

**The rule for which type to use:** does anyone need last year's report to still produce last year's numbers? If yes, Type 2. In finance, regulated industries, and anything audited, the answer is always yes.

**In your fact table, store the surrogate key that was correct at the time.** Once `fct_sale_line` holds `toy_key = 3` for that August 1st sale, the price is locked in forever and you never have to do the `BETWEEN` join again. That's the real reason surrogate keys exist.

---

# SECTION 8 — THE FAN-OUT TRAP

The bug you will meet, guaranteed, probably in your first month.

Suppose `dim_kid` accidentally has Ana twice — a bad merge, a duplicated load, whatever.

```sql
-- correct
SELECT sum(line_total) FROM fct_sale_line;
-- 194.46

-- after joining a dimension that has a duplicate row
SELECT sum(f.line_total)
FROM fct_sale_line f
JOIN dim_kid_dupe k ON k.kid_key = f.kid_key;
-- 295.94
```

**Revenue went up by joining a table that contains no revenue.**

That's fan-out. The join matched Ana's fact rows twice, so her money got counted twice. Nothing errored. The query looks perfectly reasonable. The dashboard is just wrong, by 52%.

**How to defend against it:**

1. **Test that every dimension is unique on its key.** One row per key, always.
   ```sql
   SELECT toy_key, count(*) FROM dim_toy GROUP BY 1 HAVING count(*) > 1;
   -- must return zero rows
   ```
2. **Watch your row count across joins.** If joining a dimension changes the number of fact rows, something is wrong. A dimension join should never change the count.
3. **Be suspicious when a total goes up after you add a join.** Joins should never create money.

Put both of those checks in your test suite. This is exactly the kind of thing `pytest` is for, which is why it's sitting in Module 01 of your roadmap.

---

# SECTION 9 — LAYERS

Real pipelines have floors. Names differ by shop, but the shape is always the same.

| Layer | Also called | What lives here | Rule |
|---|---|---|---|
| **Raw** | landing, bronze | Exact copy of the source. Ugly column names and all. | Never edit. Append only. |
| **Staging** | silver, prep | Cleaned and typed. One model per source table. Renamed columns, cast types, dedupe. | No joins between sources. No business logic. |
| **Core** | marts, gold, warehouse | Your star schema. Facts and dimensions. | Business logic lives here, once. |
| **Reporting** | presentation | Flat wide tables for one dashboard or team. | Cheap, disposable, rebuild freely. |

**Why raw is untouchable.** When you find a bug three months from now, you need to reprocess. If you cleaned data on the way in and threw away the original, you can't. Storage is cheaper than re-extracting from a source system that may not even have the history anymore.

**Why staging exists.** It's the only place allowed to know that the source calls it `cust_nm_1`. Everything downstream sees `customer_name`. When the source system changes, you fix one model.

**Why the reporting layer is disposable.** It's shaped for one audience. When the dashboard changes, you rewrite it. That's fine, because the core layer beneath it didn't move.

The discipline that matters: **business logic lives in exactly one layer.** If "active customer" is defined in three different places, they will drift apart, and reconciling them is a miserable week.

---

# SECTION 10 — SNOWFLAKE SPECIFICS

Things that differ from what textbooks assume, and that will come up in your new role.

**Primary and foreign keys are not enforced.** You can declare them, and Snowflake will happily let you insert duplicates. Declare them anyway — the optimiser uses them as hints and they document intent — but **your tests are the only real enforcement.** This surprises people from a Postgres or SQL Server background.

**No indexes. Micro-partitions instead.** Snowflake automatically chops tables into ~16MB chunks and records the min/max of each column in each chunk. When you filter on a date, it skips every chunk whose range can't match. This is called pruning, and it's why filtering on a well-distributed column is fast without you doing anything.

**Clustering keys, sparingly.** On very large tables (think hundreds of GB) where the natural insert order doesn't match how people filter, you can define a clustering key so Snowflake physically reorganises the data. It costs credits continuously. Don't add one until you've measured a real problem.

**Wide dimensions are genuinely fine.** Columnar storage means a 100-column dimension costs nothing to query if you select 5 columns. This is the technical reason to prefer flat wide dimensions over snowflaked ones.

**Streams and Tasks are your SCD2 machinery.** A Stream tracks what changed in a table since you last looked; a Task runs on a schedule. Together they're how you incrementally maintain Type 2 dimensions without full reloads. Worth connecting to your Snowflake study — it's the same topic from two angles.

**Zero-copy clone for testing.** `CREATE TABLE dev_fct_sales CLONE prod_fct_sales;` gives you a full copy instantly, at no storage cost until you change something. Use it to test a model change against real data volumes. There's no equivalent in most warehouses and people underuse it.

**Time Travel is not history.** It lets you query a table as it was up to 90 days ago, for recovering from mistakes. It is *not* a substitute for Type 2 dimensions — it tracks what your table said, not what was true in the business. Confusing these two is a common interview trip-up.

---

# SECTION 11 — THE RECIPE

When someone hands you a new source and says "model this," do these six steps in order. Kimball's four-step process with two additions that matter in practice.

**1. Pick the business process.** Not the table — the *event*. "A toy is sold." "An invoice is paid." "A shipment leaves." One process, one fact table.

**2. Declare the grain, out loud, as a sentence.** "One row is one product on one receipt." Write it in a comment at the top of the model. If you can't say it in one sentence, you don't understand the process yet, and no amount of SQL will fix that.

**3. List the dimensions.** Everything you'd say after "by" or "for": by toy, by kid, by shop, by day. Each becomes a dimension.

**4. List the facts.** The numbers that make sense at that grain. Quantity and line total make sense per line. Total receipt value does not — it belongs at a coarser grain.

**5. Decide what changes and how you'll handle it.** For each dimension: does anyone need history? Type 1 or Type 2? Deciding this later means a backfill.

**6. Write the tests before you write the model.** Grain uniqueness, dimension key uniqueness, no orphan foreign keys, row counts stable across joins. Four tests, and they catch most of what goes wrong.

Steps 2 and 6 are the ones people skip, and they're the ones that cost weeks.

---

# SECTION 12 — MISTAKES

**Mixing grains in one table.** Line-level rows sitting alongside order-level rows. Every aggregate is wrong forever. This is number one for a reason.

**Modelling the source system instead of the business.** If your warehouse mirrors the app's tables, you've built a slow read replica, not a model. The app is shaped for writing one row fast; the warehouse is shaped for reading a billion rows.

**Putting descriptions in the fact table.** `toy_category` in a billion-row fact table means renaming a category is a billion-row update.

**Storing averages.** Store `sum_revenue` and `count_orders`. An average of averages is not the average, and someone will eventually build a dashboard on it.

**No date dimension.** You will regret it the first time someone asks for fiscal quarters or excludes public holidays.

**Deciding SCD type after loading history.** Going Type 1 → Type 2 later means the history you needed is gone. When in doubt, and the dimension is small, go Type 2.

**Trusting declared keys in Snowflake.** They're documentation, not constraints. Test them.

**Building a star schema for one dashboard.** Sometimes the right model is one flat table. Modelling is a cost; spend it where many people benefit.

---

# DRILLS

Work in `~/modelling`. Try each for fifteen minutes before checking the solutions.

**1.** Write the grain of `raw_sales` as one sentence. Then prove it with a `count(*)` vs `count(DISTINCT ...)` check.

**2.** Explain in one sentence why `SELECT count(*) FROM raw_sales` is the wrong answer to "how many sales did we have?" Write the query that gives the right answer.

**3.** For each column in `raw_sales`, say whether it belongs in a fact table or a dimension, and which dimension.

**4.** Build `dim_kid` with a surrogate key. Then notice the problem: Ben appears with two different cities. Decide what `dim_kid` should contain and defend the choice in a comment.

**5.** Build a `dim_date` covering every day from 2026-01-01 to 2026-12-31, not just the days with sales. Include `date_key`, `full_date`, `day_name`, `month_no`, `quarter`, `is_weekend`.

**6.** Build `fct_sale_line` joining to all four dimensions. Verify it has exactly 6 rows and that `sum(line_total)` matches the raw table.

**7.** Write the revenue-by-category query against your star. Then write the same query against `raw_sales`. Compare how each would change if categories got renamed.

**8.** Using a left join from `dim_toy`, find toys that sold nothing on 2026-08-03. Explain why `raw_sales` alone cannot answer this.

**9.** Classify each measure as additive, semi-additive, or non-additive: `quantity`, `line_total`, `unit_price`, and a hypothetical `stock_on_hand`.

**10.** Build `dim_kid_scd2` where Ben moves from Toronto to Mississauga on 2026-08-03. Include `valid_from`, `valid_to`, `is_current`.

**11.** Using your SCD2 kid dimension, write the point-in-time join showing which city each sale should be attributed to. Then write the current-view query. Note where they disagree.

**12.** Deliberately duplicate a row in `dim_toy`, join it to the fact table, and show that revenue inflates. Then write the test that would have caught it.

**13.** Write four tests as SQL that each return zero rows when healthy: grain uniqueness, dimension key uniqueness, no orphan `toy_key` in the fact table, and no negative quantities.

**14.** Capstone. Add returns to the shop: a kid can bring a toy back. Design it. Decide whether it's a new fact table or negative rows in the existing one, declare the grain, and defend the choice in a comment. Then write the query for net revenue.

---

# SOLUTIONS

## 1. Grain

> **One row is one toy on one receipt.**

```sql
SELECT count(*) AS n_rows,
       count(DISTINCT (receipt_no, toy_name)) AS n_unique
FROM raw_sales;
-- 6, 6  ✓
```

If those two numbers ever diverge, either you have duplicate loads or the same toy appears twice on one receipt — and if the latter is legitimate, your grain is actually "one *line* on one receipt" and you need a line number.

## 2. Counting sales

`count(*)` counts *lines*, not sales. Ana's receipt R-001 is two lines but one sale.

```sql
SELECT count(DISTINCT receipt_no) AS sales FROM raw_sales;
-- 5
```

This is the grain mistake in its smallest possible form, and it's the same mistake that produces wrong revenue in real systems.

## 3. Column placement

| Column | Goes in |
|---|---|
| `receipt_no` | fact — degenerate dimension |
| `sold_at` | fact, as `date_key` → `dim_date` |
| `kid_name` | `dim_kid` |
| `kid_city` | `dim_kid` |
| `toy_name` | `dim_toy` |
| `toy_category` | `dim_toy` |
| `toy_price` | fact as `unit_price`; also `dim_toy` as `list_price` |
| `quantity` | fact |
| `shop_name` | `dim_shop` |

Price appearing in both is deliberate, not a mistake. `dim_toy.list_price` is what the toy *costs today*. `fct_sale_line.unit_price` is what was *actually charged*, which may differ because of a discount. Keeping both lets you answer "how much discount did we give?"

## 4. dim_kid, and the Ben problem

```sql
SELECT kid_name, count(DISTINCT kid_city) AS cities
FROM raw_sales GROUP BY 1 HAVING count(DISTINCT kid_city) > 1;
-- Ben, 2
```

```sql
-- Grain: one row = one kid (current state).
-- Ben appears in two cities. Two possibilities: he moved, or one is a typo.
-- We cannot tell from this data, which is itself the finding.
-- Taking the most recent value is the pragmatic Type 1 choice; if anyone
-- needs "where did Ben live when he bought this", this must become Type 2.
CREATE OR REPLACE TABLE dim_kid AS
SELECT
  row_number() OVER (ORDER BY kid_name) AS kid_key,
  kid_name,
  kid_city
FROM (
  SELECT kid_name, kid_city,
         row_number() OVER (PARTITION BY kid_name ORDER BY sold_at DESC) AS rn
  FROM raw_sales
) WHERE rn = 1;
```

The real answer in a job is: **go ask someone.** "Does Ben have two rows because he moved?" is a business question, not a SQL question, and guessing is how bad models get built.

## 5. Full date dimension

```sql
CREATE OR REPLACE TABLE dim_date AS
SELECT
  CAST(strftime(d, '%Y%m%d') AS INTEGER) AS date_key,
  CAST(d AS DATE)                        AS full_date,
  dayname(d)                             AS day_name,
  CAST(strftime(d, '%m') AS INTEGER)     AS month_no,
  CAST(strftime(d, '%Y') AS INTEGER)     AS year_no,
  quarter(d)                             AS quarter_no,
  dayofweek(d) IN (0, 6)                 AS is_weekend
FROM generate_series(DATE '2026-01-01', DATE '2026-12-31', INTERVAL 1 DAY) AS t(d);
```

365 rows. Generating the full year rather than only days with sales is the point: a report for a quiet week should show zeros, not missing rows. In Snowflake the equivalent generator is `GENERATOR(ROWCOUNT => n)` with `DATEADD`.

## 6. The fact table

```sql
CREATE OR REPLACE TABLE fct_sale_line AS
SELECT
  s.receipt_no,
  CAST(strftime(s.sold_at,'%Y%m%d') AS INTEGER) AS date_key,
  k.kid_key, t.toy_key, sh.shop_key,
  s.quantity,
  s.toy_price              AS unit_price,
  s.quantity * s.toy_price AS line_total
FROM raw_sales s
JOIN dim_kid  k  ON k.kid_name   = s.kid_name
JOIN dim_toy  t  ON t.toy_name   = s.toy_name
JOIN dim_shop sh ON sh.shop_name = s.shop_name;

SELECT count(*) AS n, sum(line_total) AS revenue FROM fct_sale_line;
-- 6, 194.46
```

Always compare the fact row count and total against the source. An `INNER JOIN` to a dimension that's missing a row will silently drop facts, and that's a bad way to lose money.

## 7. Revenue by category

```sql
-- star
SELECT t.toy_category, sum(f.line_total) AS revenue
FROM fct_sale_line f JOIN dim_toy t USING (toy_key)
GROUP BY 1 ORDER BY revenue DESC;

-- raw
SELECT toy_category, sum(quantity * toy_price) AS revenue
FROM raw_sales GROUP BY 1 ORDER BY revenue DESC;
```

Both return Robots 104.96, Puzzles 47.50, Outdoor 42.00.

The difference is what happens on a rename. In the star, `UPDATE dim_toy SET toy_category = 'Robotics' WHERE toy_category = 'Robots'` touches one row. Against raw, it touches every fact row that ever mentioned robots — and if the load is append-only, you can't update history at all.

## 8. Toys that sold nothing

```sql
SELECT t.toy_name
FROM dim_toy t
LEFT JOIN fct_sale_line f
  ON f.toy_key = t.toy_key AND f.date_key = 20260803
WHERE f.toy_key IS NULL;
-- Kite, Puzzle Cube
```

`raw_sales` can't answer it because **absence has no row.** A toy that sold nothing simply doesn't appear. The dimension is the complete list of things that exist, so it's the only place a "what didn't happen" question can start. This is the strongest single argument for dimensions.

## 9. Measure types

| Measure | Type | Why |
|---|---|---|
| `quantity` | additive | sum across any dimension |
| `line_total` | additive | same |
| `unit_price` | non-additive | summing prices is meaningless; average it, weighted by quantity |
| `stock_on_hand` | semi-additive | sums across shops, but not across days — take the latest |

The weighted average matters: `sum(line_total) / sum(quantity)`, not `avg(unit_price)`. The second treats a 1-unit sale and a 400-unit sale as equally important.

## 10. SCD2 kid dimension

```sql
CREATE OR REPLACE TABLE dim_kid_scd2 (
  kid_key INTEGER, kid_name VARCHAR, kid_city VARCHAR,
  valid_from DATE, valid_to DATE, is_current BOOLEAN
);

INSERT INTO dim_kid_scd2 VALUES
 (1,'Ana','Brampton',    DATE '2026-01-01', DATE '9999-12-31', TRUE),
 (2,'Ben','Toronto',     DATE '2026-01-01', DATE '2026-08-02', FALSE),
 (3,'Ben','Mississauga', DATE '2026-08-03', DATE '9999-12-31', TRUE),
 (4,'Cara','Toronto',    DATE '2026-01-01', DATE '9999-12-31', TRUE);
```

The windows must not overlap and must not have gaps. `valid_to` of the old row is the day before `valid_from` of the new one. A test for overlapping windows per natural key is worth writing.

## 11. Point in time vs current

```sql
-- what was true at the time of sale
SELECT s.receipt_no, s.kid_name, k.kid_city AS city_then
FROM raw_sales s
JOIN dim_kid_scd2 k
  ON k.kid_name = s.kid_name
 AND CAST(s.sold_at AS DATE) BETWEEN k.valid_from AND k.valid_to
WHERE s.kid_name = 'Ben';
-- R-002 → Toronto,  R-005 → Mississauga

-- what is true now
SELECT s.receipt_no, k.kid_city AS city_now
FROM raw_sales s
JOIN dim_kid_scd2 k ON k.kid_name = s.kid_name AND k.is_current
WHERE s.kid_name = 'Ben';
-- R-002 → Mississauga,  R-005 → Mississauga
```

They disagree on R-002, and **both are correct answers to different questions.** "How much did we sell in Toronto in August?" wants the first. "How much have our current Mississauga customers ever spent?" wants the second. Knowing which one a stakeholder means is most of the job.

## 12. Fan-out

```sql
CREATE OR REPLACE TABLE dim_toy_dupe AS
SELECT * FROM dim_toy UNION ALL SELECT * FROM dim_toy WHERE toy_name = 'Red Robot';

SELECT sum(f.line_total) FROM fct_sale_line f
JOIN dim_toy_dupe t ON t.toy_key = f.toy_key;
-- inflated: Red Robot's rows counted twice
```

The catch:

```sql
SELECT toy_key, count(*) AS n
FROM dim_toy_dupe GROUP BY 1 HAVING count(*) > 1;
```

Run that on every dimension, every load. In dbt it's the built-in `unique` test; in plain SQL it's four lines.

## 13. Four tests

```sql
-- 1. grain uniqueness on the fact table
SELECT receipt_no, toy_key, count(*)
FROM fct_sale_line GROUP BY 1,2 HAVING count(*) > 1;

-- 2. dimension key uniqueness
SELECT toy_key, count(*) FROM dim_toy GROUP BY 1 HAVING count(*) > 1;

-- 3. no orphan foreign keys
SELECT f.toy_key FROM fct_sale_line f
LEFT JOIN dim_toy t USING (toy_key) WHERE t.toy_key IS NULL;

-- 4. no negative quantities
SELECT * FROM fct_sale_line WHERE quantity <= 0;
```

All four return zero rows when healthy. That's the convention — a test that returns rows has found problems, and the rows *are* the problem list. Wrap each in a pytest assertion and you've got the beginnings of a real data quality suite.

## 14. Capstone — returns

```sql
-- DECISION: separate fact table, not negative rows in fct_sale_line.
--
-- Grain: one row = one toy returned on one return slip.
--
-- Why separate:
--   * Different event, different date. A toy sold on the 1st and returned
--     on the 10th belongs to two different days. Negative rows would force
--     one date onto both, and any daily revenue figure would be wrong.
--   * Different attributes. Returns have a reason code; sales don't.
--   * Different grain risk. A return can be partial, and quantities returned
--     don't have to match quantities sold.
--   * Sales stay immutable. What we sold on the 1st never changes.
--
-- Negative rows would be acceptable only if returns were always same-day
-- and carried no extra attributes — rare in practice.

CREATE OR REPLACE TABLE fct_return_line (
  return_no    VARCHAR,
  receipt_no   VARCHAR,    -- links back to the original sale
  date_key     INTEGER,    -- date of the RETURN
  kid_key      INTEGER,
  toy_key      INTEGER,
  shop_key     INTEGER,
  quantity     INTEGER,    -- positive; the sign lives in the semantics
  refund_total DECIMAL(10,2),
  reason_code  VARCHAR
);

INSERT INTO fct_return_line VALUES
 ('RT-01','R-001', 20260805, 1, 3, 2, 1, 24.99, 'FAULTY');
```

Net revenue:

```sql
WITH sales AS (
  SELECT date_key, toy_key, sum(line_total) AS gross
  FROM fct_sale_line GROUP BY 1,2
),
refunds AS (
  SELECT date_key, toy_key, sum(refund_total) AS refunded
  FROM fct_return_line GROUP BY 1,2
)
SELECT
  coalesce(s.toy_key, r.toy_key) AS toy_key,
  coalesce(sum(s.gross), 0)      AS gross_revenue,
  coalesce(sum(r.refunded), 0)   AS refunds,
  coalesce(sum(s.gross), 0) - coalesce(sum(r.refunded), 0) AS net_revenue
FROM sales s
FULL OUTER JOIN refunds r ON s.toy_key = r.toy_key AND s.date_key = r.date_key
GROUP BY 1 ORDER BY net_revenue DESC;
```

`FULL OUTER JOIN` matters here: a day with returns but no sales still needs a row, otherwise refunds silently vanish from your totals. Two fact tables sharing the same dimensions is called a **fact constellation**, and it's the normal shape of a real warehouse.

---

## When you get stuck

1. Say the grain out loud. Most confusion is a grain problem wearing a disguise.
2. If a total looks wrong after adding a join, check the dimension for duplicate keys first.
3. If a number changed and you don't know why, check whether the dimension is Type 1 — history may have been overwritten.
4. `count(*)` before and after every join. Fact row counts should not move.
5. Still stuck — paste the model and the numbers and I'll work through it with you.
