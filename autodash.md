# AutoDash — prompts to build it with Qwen

Nov 24, 2026 · @Onkar Suryawanshi

## How to use these prompts

Build AutoDash in 14 small steps: paste the master prompt at the start of a chat, then send one step prompt per message. Each step makes one file and ends with a check you run before going on.

1. **Make the folders first:** `autodash/`, with `engine/`, `components/arrange/`, `tests/` and `.streamlit/` inside it. Save each answer under the exact path its step names.
2. **Start every chat with the master prompt** (next section). It holds the rules and every module's contract (names, inputs, outputs), so later steps work without pasting earlier files. Start a new chat every two or three steps; long chats make models forget the rules.
3. **Send the steps in order.** Steps 1 to 7 give a working app (upload, cleaning, dashboard and findings). Steps 8 to 14 add features, and each relies on the contracts of the steps before it.
4. **Run the step's check before the next step.** If it fails, send the fix-it prompt with the error (section 5). Don't build on a file that fails its check.
5. **Settings:** temperature 0.2 to 0.3, the longest answer length allowed. If your Qwen has a thinking mode, keep it on for steps 3 to 6 (the cleaning and analysis logic) and turn it off when a long file keeps getting cut off.
6. **When an answer stops mid-file,** send: `Continue exactly where you stopped, from the line that starts with <paste the last line>. Do not repeat anything.` Then join the two parts. Steps with big files already ask for them in parts.
7. **Smaller models (under 14B):** send one part per message even where a step allows two, and read each function name against the contract before saving.

## Master prompt (paste first in every chat)

This prompt fixes the rules and every file's contract, so each step can be built on its own. Paste it whole; the steps refer to its section names.

```text
You are a senior Python developer. We are building AutoDash: a Streamlit app where a user
uploads a data file and the app reads it, cleans it, analyses it and shows a dashboard,
a data explorer and exports. I send one build step per message. Each step asks for one
file (or one part of a file).

WHERE IT RUNS
- An offline Windows work computer: no internet, no CDN, no telemetry, no web fonts.
- Python 3.10 to 3.13. Packages: streamlit>=1.40, pandas>=2.1, numpy>=1.26, plotly>=5.18,
  openpyxl, pyarrow, duckdb>=1.0. Optional (import inside try/except, and say on screen
  which package is missing): lxml, python-pptx, matplotlib.
- Must work on Streamlit 1.40 and newer. Arguments that newer versions added
  (width="stretch", vertical_alignment, border= on st.metric, type="tertiary") are passed
  through uikit.call(), which drops keyword arguments the installed version rejects.

RULES FOR EVERY ANSWER
1. Start with one line "File: <path>", then the COMPLETE file in one code block. Nothing
   after it. No "...", no "rest unchanged", no TODO, no placeholder.
2. Use the names and arguments in CONTRACTS exactly. Extra helpers are private (_name).
3. engine/ files never import streamlit: plain Python a test can call. Top-level files do
   the Streamlit work and keep logic in engine/.
4. One failing chart or section shows a short message in its place; the page goes on.
5. Words on screen: plain English, short sentences, what happened and what to do next.
6. Safety: never eval/exec data; no pickle; XML without external entities; SQL is
   read-only (only SELECT or WITH; refuse read_csv, read_parquet, read_json, COPY, ATTACH,
   INSTALL, LOAD, PRAGMA, EXPORT).
7. Never read a file over 1 GB into pandas. Those go through DuckDB (step 10).
8. Type hints, short functions, a docstring on each public one. st.cache_data only in UI.
9. Before answering, check that every name you use is defined or imported, and that each
   contract function exists with the right arguments.
10. Inside one function, never reuse a variable name for a different kind of value.
11. Widget keys are unique: build them from the table name and the item's id.

FILES
app.py uikit.py dashboard_view.py explorer_view.py bigfile_view.py combine_view.py
transactions_view.py ask_view.py requirements.txt .streamlit/config.toml
components/arrange/index.html tests/
engine/__init__.py samples.py loader.py cleaner.py profiler.py layout.py planner.py
       charts.py insights.py sql.py join.py bigfile.py bigdata.py txn.py slides.py llm.py ask.py

CONTRACTS
engine/samples.py  SAMPLES: dict label -> key; get_sample(key) -> (file_name, bytes)
engine/loader.py   @dataclass ReadOptions(delimiter=None, encoding=None, header="auto")
                   @dataclass LoadedTable(name, df, source_format, notes=list)
                   class LoadError(Exception)   # the message is shown to the user
                   load_file(file_name, data: bytes, options=None) -> list[LoadedTable]
engine/cleaner.py  @dataclass CleanOptions(remove_duplicates=True, fix_types=True,
                     unify_labels=True, remove_junk_rows=True, drop_empty=True,
                     missing="keep", outliers="flag", dayfirst=True, enabled=True)
                     classmethods types_only(dayfirst=True) and off()
                   @dataclass Action(category, title, detail, column=None, affected=0)
                   @dataclass CleanResult(df, actions, warnings, shape_before,
                     health_before, health_after)
                   clean_dataframe(raw, options=None) -> CleanResult
engine/profiler.py @dataclass ColumnInfo(name, role, kind, n_missing, missing_pct,
                     n_unique, agg=None, flag_true=None)
                     role: measure|category|date|flag|id|text  kind: number|text|date|bool
                     agg: sum|mean (measures only)
                   @dataclass Profile(n_rows, n_cols, columns, measures, categories, dates,
                     flags, ids, texts, primary_date, primary_measure); info(col)
                   profile_dataframe(df) -> Profile
engine/layout.py   @dataclass Tile(id, kind, x=None, y=None, color=None, agg="sum",
                     size="half", title=""); to_dict(); classmethod from_dict(d)
                     kind: trend|bar|parts|rate|histogram|scatter|heatmap|table
                   new_id(*parts) -> str; title_of(tile) -> str
                   plan_tiles(df, prof) -> list[Tile]
engine/planner.py  @dataclass KPI(label, value, unit="", delta=None, delta_text="",
                     help="", key=""); display() -> str
                   kpi_options(df, prof) -> list[(key, label)]
                     keys: rows, sum:<col>, mean:<col>, rate:<col>, distinct:<col>, top:<col>
                   default_kpi_keys(df, prof) -> list[str]   (at most 5)
                   build_kpis(df, prof, keys) -> list[KPI]   (skips keys it can't compute)
engine/charts.py   class ChartError(Exception)
                   aggregate(df, prof, tile) -> DataFrame   (the small table a tile plots)
                   draw(agg, prof, tile, theme="light") -> Figure
                   make_chart(df, prof, tile, theme="light") -> Figure   (draw of aggregate)
engine/insights.py @dataclass Insight(title, detail, score, kind)
                   generate_insights(df, prof, limit=10) -> list[Insight]
engine/sql.py      class SQLError(Exception); @dataclass QueryResult(df, seconds, engine)
                   quote(name) -> str; check_read_only(query)   (raises SQLError)
                   run_query(query, tables) -> QueryResult
                     (tables: dict name -> function returning a DataFrame)
                   class TableFacts(df)
                   class Example(NamedTuple): title, sql, output, rows, columns
                   example_queries(df, engine, facts=None) -> list[Example]
engine/join.py     class JoinError(ValueError)
                   suggest_keys(left, right) -> list[(left_col, right_col, score)]
                   join_tables(left, right, left_on: str, right_on: str, how="left") -> (df, notes)
                   stack_tables(frames: dict name -> df, add_source=True) -> (df, notes)
engine/bigfile.py  class BigFileError(Exception); available() -> bool; cache_root() -> str
                   kind_of(name) -> "csv"|"parquet"|"json"|None
                   json_shape(head: bytes, whole=False) -> "lines"|"array"|"document"|""
                   sniff(path) -> FileInfo(path, name, size, mtime, kind, json_format, bom)
                   start(path, dayfirst=True) -> Job
                   Job: state (running|done|failed|stopped), progress, message, result, stop()
                   Prepared(key, parquet, rows, columns, notes, file)   (job.result)
engine/bigdata.py  class BigData(prepared): rows, columns, query(sql) -> QueryResult,
                     sample(n=200_000, where=""), head(n=1000, offset=0), profile(),
                     aggregate(prof, tile, where=""), big_kpis(prof, keys, where=""), facts()
engine/txn.py      guess_roles(df, prof) -> Roles(date, amount, sender, receiver)
                   findings(df, roles, threshold=10000) -> list[Finding(title, detail, rows)]
                   party_summary(df, roles, party) -> dict
engine/slides.py   figures: list of (title, Figure); each function returns the file as bytes
                   deck_pptx(title, kpis, insights, figures)
                   report_pdf(title, kpis, insights, figures)
                   dashboard_html(title, kpis, insights, figures)
engine/llm.py      @dataclass LLMConfig(url, model, key="", style="openai"); class LLMError
                   from_env() -> LLMConfig|None; chat(cfg, system, messages) -> str
                   values_allowed() -> bool
engine/ask.py      answer(question, df, prof, cfg=None) -> Answer(text, table, figure, sql)
uikit.py           call(fn, *args, **kw); icon(name) -> str; card(key) -> container
                   segmented(label, options, key) -> str; CSS: str; footer()
                   @dataclass(eq=False) Source(key, name, data=None, df=None, path=None):
                     one file: an upload or a sample (data: bytes), a table made in
                     Combine (df), or a file opened by path (path)
views (optional: app.py imports each, and engine.slides, inside try/except; a missing file
hides its feature, a file that fails to import shows its error)
  explorer_view.render_explorer(tables, name)
    tables: dict table name -> function returning its cleaned DataFrame; name: shown first
  dashboard_view.current_layout(uid, df, prof) -> (tiles, kpi_keys, edited)
    customise_panel(uid, tiles, kpi_keys, df, prof, kpis)
    draw_kpis(kpis, edit, uid, tiles, kpi_keys)
    draw_tiles(df, prof, tiles, edit, uid, figure=None)   (figure(tile) -> Figure)
  bigfile_view.wants(src) -> bool; render(src); sidebar_box(); opened() -> Source|None;
    storage_settings(); start_page_card()
  combine_view.render_combine(sources, tables_of)
    tables_of(src) -> dict table name -> cleaned DataFrame. A table made here is kept in
    st.session_state["combined"][name] (a DataFrame), and app.py lists it as a file.
  transactions_view.render_transactions(df, prof)
  ask_view.render_ask(df, prof)
```

## Core build steps (1 to 7): a working app

After step 7 you can upload a file, get it cleaned, and see KPIs, charts, findings and a cleaning report. Each step lists its prompt, then a check to run from the `autodash` folder.

### Step 1: setup files and sample data

Type these three files yourself; they are short and must be exact.

`requirements.txt`

```text
streamlit>=1.40
pandas>=2.1
numpy>=1.26
plotly>=5.18
openpyxl>=3.1
pyarrow>=14
duckdb>=1.0
lxml>=4.9
python-pptx>=0.6.21
matplotlib>=3.5
```

`.streamlit/config.toml`

```text
[server]
maxUploadSize = 10240

[browser]
gatherUsageStats = false

[client]
toolbarMode = "minimal"

[theme]
primaryColor = "#00965A"
backgroundColor = "#FFFFFF"
secondaryBackgroundColor = "#F1F6F3"
textColor = "#141414"
```

`engine/__init__.py`

```text
"""AutoDash engine: reading, cleaning and analysing tables, without Streamlit."""
```

Then send:

```text
STEP 1. Write engine/samples.py: messy, fictional demo files for testing the app. Only numpy,
pandas and openpyxl. Fixed random seeds, so each file is the same on every run.
SAMPLES = {"Retail sales (messy CSV)": "retail", "HR attrition (Excel, 2 sheets)": "hr",
           "Website traffic (nested JSON)": "web", "Transfers (patterns to find)": "transfers"}
get_sample(key) -> (file_name, bytes)
- retail: retail_sales_messy.csv, 3000 orders over 2023-2024. Columns: Order ID (ORD-10001...),
  Order Date, Region, City, Category, Sub-Category, Segment, Sales, Quantity, Discount, Profit,
  Returned. Plant: two title lines above the header; Sales as text like "Rs 1,234.50"; dates in
  three formats (2024-03-05, 05/03/2024, 5 Mar 2024); city typos (Mumbia for Mumbai) in 1% of
  rows; Region in mixed case with spaces (" west", "WEST"); 40 exact duplicate rows; a last row
  "Total" with sums; 2% of Discount as "N/A"; Returned as Yes/No. Sales grows about 35% over the
  two years with a peak in October and November.
- hr: hr_attrition.xlsx with sheets "Employees" (1200 rows) and "Notes" (5 text rows). A title
  row above the header; Employee ID as text with leading zeros ("00123"); Department with both
  "R & D" and "R&D"; Monthly Income; Join Date as real Excel dates; some "#N/A" cells;
  Attrition Yes/No.
- web: web_traffic.json = {"site": "example", "records": [...]}, 60 days x 24 hours. Each record:
  {"timestamp", "page", "visits", "metrics": {"bounce_rate": "43.2%", "avg_seconds": 51},
  "tags": ["campaign", "mobile"]}.
- transfers: transfers_demo.csv, 9000 transfers in 2024 among about 2,000 accounts (ACC-0001
  ...): Date, Sender, Receiver, Amount, Currency, Channel. Plant: a circle A->B->C->A of
  similar amounts within 10 days; one sender with 12 payments between 9,000 and 9,990; one
  account that sends on 95% of what it receives within a day; one sender with 20 payments,
  all round thousands.
```

Check:

```text
python -c "from engine.samples import SAMPLES, get_sample; [print(get_sample(k)[0], len(get_sample(k)[1])) for k in SAMPLES.values()]"
```

### Step 2: the file reader

```text
STEP 2. Write engine/loader.py (see CONTRACTS). load_file returns one LoadedTable per table
in the file, or raises LoadError with a plain message.
Decide the format from the content first, then the extension:
- CSV, TSV, TXT: encoding from the BOM, else utf-8, else cp1252, else latin-1. Separator by
  csv.Sniffer on the first 64 KB among , ; tab |. Read every column as text (dtype=str,
  keep_default_na=False): the cleaner decides types later.
- Header detection (header="auto"): skip title lines above the real header. The header is the
  first of the first 20 rows whose cells are mostly filled text and whose next rows have the
  same number of filled cells. Make names unique (Name, Name_2); blank names become Column_3.
- Excel .xlsx/.xlsm: each sheet with data is a table named after the sheet; same header
  detection; values as read (dates stay dates).
- JSON: a list of records -> one table (pandas.json_normalize, nested keys joined by ".",
  lists of plain values joined as "a, b"); an object holding lists of records -> one table per
  list, named by its key path; JSON Lines (one object per line) -> one table. Strip a BOM first.
- Parquet: pandas.read_parquet.
- XML: lxml with resolve_entities=False and no_network=True; the most repeated element is a
  row; its child elements and attributes are columns.
- .gz: decompress, then read by the inner name.
ReadOptions delimiter, encoding and header override the detection when set.
Add notes like "Skipped 2 title rows above the header." to LoadedTable.notes.
Refuse files over 1 GB: LoadError("<name> is over 1 GB: it is read with DuckDB (big files).").
```

Check (expect retail about 3,041 rows with a note on 2 title rows, hr with two sheets, web with a `metrics.bounce_rate` column):

```text
python -c "from engine.loader import load_file; from engine.samples import get_sample; [print(t.name, t.df.shape, t.notes) for k in ('retail','hr','web','transfers') for t in load_file(*get_sample(k))]"
```

### Step 3: the cleaner (two parts)

```text
STEP 3, PART 1 of 2. Write the first half of engine/cleaner.py: imports, CleanOptions, Action,
CleanResult (CONTRACTS) and these private steps. Each takes (df, opt, actions) and returns
the new DataFrame:
- _trim: strip spaces, collapse runs of spaces; "", NA, N/A, null, None, -, #N/A, nan become
  missing. Action("Values", "Marked 37 cells like N/A as missing", ...).
- _numbers: a text column becomes numbers when 90%+ of its filled cells parse after removing
  currency signs and codes (Rs, INR, $, EUR, GBP), thousands commas, spaces and a trailing %
  (45% -> 45); (1,234) -> -1234. Keep codes as text: leading zeros (00123) or names ending in
  id, code, zip, phone, account.
- _dates: a text column becomes datetime when 90%+ parse with pandas.to_datetime(format="mixed",
  dayfirst=opt.dayfirst); say how many cells did not parse.
- _booleans: columns whose two values are Yes/No, Y/N, True/False or 1/0 become True/False.
Every change adds one Action(category, title, detail, column, affected).
Stop after these functions. Part 2 continues the same file.
```

```text
STEP 3, PART 2 of 2. Continue engine/cleaner.py (no imports again):
- _junk_rows: drop rows that repeat the header, rows whose first filled text cell is Total,
  Grand total or Subtotal (any case), and rows with one filled cell in a table of 4+ columns.
- _labels: in text columns with at most 500 distinct values, map variants that differ only by
  case, spaces or punctuation to the most common spelling ("WEST", " west" -> "West"). Then fix
  typos: a value under 2% as frequent as a common value with difflib ratio >= 0.85 takes the
  common spelling ("Mumbia" -> "Mumbai"). Never merge two common values.
- _duplicates: drop exact duplicate rows. _empty: drop rows and columns with no values.
- _missing: "keep"; "fill" (median for numbers, most common value for text); "drop" rows.
- _outliers: beyond 3 x IQR in a number column: "flag" adds a bool column "<col> outlier",
  "cap" clips, "remove" drops rows, "none" does nothing.
- _health(df) -> int 0..100: 100 minus 40 x missing share, minus 20 x duplicate share, minus
  20 x share of text cells in columns that are mostly numbers or dates.
- clean_dataframe(raw, options=None): work on a copy; run trim, junk rows, numbers, dates,
  booleans, labels, duplicates, empty, missing, outliers; each step catches its own error and
  adds a warning instead. enabled=False returns the data as it came, with no actions.
  types_only(): trim, numbers, dates, booleans only. off(): enabled=False.
```

Check (expect about 3,000 rows, Sales as numbers, Order Date as dates, 4 regions, health after higher than before):

```text
python -c "from engine.loader import load_file; from engine.cleaner import clean_dataframe; from engine.samples import get_sample; r=clean_dataframe(load_file(*get_sample('retail'))[0].df); print(r.df.shape, r.health_before, r.health_after); print(r.df.dtypes); print(r.df['Region'].value_counts()); [print(a.category, a.title) for a in r.actions]"
```

### Step 4: column roles

```text
STEP 4. Write engine/profiler.py (CONTRACTS). profile_dataframe(df) gives each column a role:
- date: datetime columns. primary_date = the date column with the most distinct days.
- flag: bool columns or two-value text like Yes/No; flag_true = the value that means yes.
- id: distinct on 95%+ of rows and named like id, code, number, ref, or values like ORD-10001.
- measure: numbers that are not ids, not years (integers 1900-2100 named year) and not codes.
  agg = "mean" for rates, percentages, prices, scores, ages and ratings (names with rate, pct, %,
  price, score, age, rating, avg), else "sum". primary_measure = the money-like measure (sales,
  revenue, amount, value, income, profit) with the biggest total, else the first "sum" measure.
- category: text with 2 to 50 distinct values. text: any other text (comments).
Fill n_missing, missing_pct (0-100) and n_unique. The lists keep column order, except
categories: fewest distinct values (2 or more) first.
```

Check (expect measures Sales, Quantity, Discount, Profit; id Order ID; flag Returned):

```text
python -c "from engine.loader import load_file; from engine.cleaner import clean_dataframe; from engine.profiler import profile_dataframe; from engine.samples import get_sample; df=clean_dataframe(load_file(*get_sample('retail'))[0].df).df; p=profile_dataframe(df); print(p.measures, p.categories, p.dates, p.flags, p.ids, p.primary_measure, p.primary_date)"
```

### Step 5: the dashboard plan and KPI tiles (two messages)

```text
STEP 5, PART 1. Write engine/layout.py (CONTRACTS). Tile.from_dict ignores unknown keys.
new_id(*parts) = first 10 hex characters of sha1 of the parts joined by "|".
title_of(t): t.title if set, else trend "Total Sales over time", bar "Total Sales by Region",
parts "Share of Sales by Segment", rate "Returned rate by Category", histogram "Distribution
of Profit", scatter "Profit against Sales", heatmap "Sales by Region and Category", table "Top
rows" ("Average" when agg is "mean", "Rows" when there is no measure).
plan_tiles(df, prof): 6 to 12 tiles, best first: trend of the primary measure over the
primary date (size "full"); bar by the best category; parts by the next category with 3 to 8
values; rate of the first flag by the best category; histogram of the primary measure; scatter
of the two biggest measures; heatmap of two categories; bars by up to 3 more categories; table.
Skip tiles whose columns are missing. id = new_id(kind, x, y, color).
```

```text
STEP 5, PART 2. Write engine/planner.py (CONTRACTS).
kpi_options: ("rows", "Rows"); per measure ("sum:<m>", "Total <m>") or ("mean:<m>", "Average <m>")
by its agg; ("rate:<flag>", "<flag> rate"); ("distinct:<col>", "Distinct <col>") for ids and
categories; ("top:<category>", "Top <category>").
default_kpi_keys: rows, the primary measure, up to two more measures, the first flag rate.
build_kpis: one KPI per key it can compute, in order; set KPI.key to the key.
- rows: len(df). sum/mean: the number. rate: the share of True. distinct: nunique.
  top: the most common value (by the primary measure's total when its agg is "sum", else by
  rows); help says its share ("35% of Sales").
- With a primary date covering 2+ months: delta = last full month against the month before,
  as a fraction; delta_text like "Dec vs Nov".
- display(): 12345 -> "12.3K", 4500000 -> "4.5M", 7.2e9 -> "7.2B"; rates as "20.0%"; text
  as it is; a unit goes in front ("Rs 1.2M").
Compute each key in its own try/except. Use a local variable name for the row count that no
other branch reuses.
```

Check:

```text
python -c "from engine.loader import load_file; from engine.cleaner import clean_dataframe; from engine.profiler import profile_dataframe; from engine.layout import plan_tiles, title_of; from engine.planner import default_kpi_keys, build_kpis; from engine.samples import get_sample; df=clean_dataframe(load_file(*get_sample('retail'))[0].df).df; p=profile_dataframe(df); print([title_of(t) for t in plan_tiles(df, p)]); print([(k.key, k.display(), k.delta_text) for k in build_kpis(df, p, default_kpi_keys(df, p) + ['top:Region'])])"
```

### Step 6: charts and findings (two messages)

```text
STEP 6, PART 1. Write engine/charts.py (CONTRACTS) with plotly.graph_objects, in two stages so
big files (step 10) can compute the same small tables in SQL.
aggregate(df, prof, tile) returns the small table a tile plots, with these columns:
- trend: period, value. tile.y by month of tile.x (sum or mean by tile.agg; a row count when y
  is None); by day when the data spans under 60 days, by year over 6 years.
- bar: label, value. The top 12 values of tile.x, the rest summed as "Other".
- parts: label, value. Up to 7 values plus "Other".
- rate: label, rate, rows. The share of tile.y being true for each value of tile.x (top 12 by
  rows, the rest as "Other").
- heatmap: row, col, value. The aggregate by tile.x and tile.color, top 12 of each.
- histogram: value (the filled values of tile.x). scatter: x, y, color (up to 5,000 sampled
  rows). table: the first 50 rows.
draw(agg, prof, tile, theme="light") turns that table into a figure: trend a line with
markers; bar horizontal, largest on top; parts a donut; rate bars with the overall rate
(weighted by rows) as a dashed line; histogram 30 bins with the median as a dashed line;
scatter points coloured by color when set; heatmap; table a plotly Table.
make_chart(df, prof, tile, theme="light") = draw(aggregate(df, prof, tile), prof, tile, theme).
Look: transparent background, margins 10, no chart title (the card shows it), number axes as
1.2K and 3.4M, colours #00965A #2A78D6 #EDA100 #E87BA4 #6250D6 #EB6834. theme="dark": light
text, grid at 15% white. Raise ChartError("Needs a number column.") and the like when a tile
can't be drawn.
```

```text
STEP 6, PART 2. Write engine/insights.py (CONTRACTS). generate_insights returns findings in
plain English, highest score (0 to 1) first. Every number is computed, never invented. Kinds:
- trend: the primary measure, first 3 months against last 3: "Sales grew 37% from Jan-Mar 2023
  to Oct-Dec 2024".
- peak: "November is the busiest month: 1.4x an average month".
- concentration: one category value over 40% of the total: "West makes 35% of Sales".
- gap: a flag rate 2x or more apart between values with 30+ rows each: "Returned rate is 2.3x
  higher for Furniture (12%) than Technology (5%)".
- correlation: two measures with |r| >= 0.6: "Profit rises with Sales (r = 0.72)".
- outliers: over 1% of a measure beyond 3 x IQR. missing: a column over 20% empty.
Each kind in its own try/except.
```

Check (after "charts ok", a small table with the columns period and value, then the findings):

```text
python -c "from engine.loader import load_file; from engine.cleaner import clean_dataframe; from engine.profiler import profile_dataframe; from engine.layout import plan_tiles; from engine.charts import make_chart, aggregate; from engine.insights import generate_insights; from engine.samples import get_sample; df=clean_dataframe(load_file(*get_sample('retail'))[0].df).df; p=profile_dataframe(df); t=plan_tiles(df, p); [make_chart(df, p, x) for x in t]; print('charts ok'); print(aggregate(df, p, t[0]).head()); [print(round(i.score, 2), i.title) for i in generate_insights(df, p)]"
```

### Step 7: the app (three messages)

```text
STEP 7, PART 1 of 3. Write uikit.py (CONTRACTS), the shared Streamlit helpers.
- call(fn, *args, **kw): drop keyword arguments that inspect.signature(fn) doesn't accept
  (keep all when it takes **kwargs). Cache signatures.
- icon(name): ":material/<name>: " on Streamlit 1.39+, else "".
- card(key): call(st.container, border=True, key="ad-card-" + key).
- segmented(label, options, key): st.segmented_control when it exists (first option when
  nothing is picked), else st.radio(horizontal=True).
- CSS: a <style> block: rounded cards, metric value 1.9rem, a muted class "ad-note"
  (font-size .8rem, opacity .75). No external fonts.
- footer(): a fixed line at the bottom: "This app can make mistakes. Please double check responses."
- @dataclass(eq=False) Source(key: str, name: str, data: bytes | None = None, df: object =
  None, path: str | None = None): one file the app can open (see CONTRACTS).
```

```text
STEP 7, PART 2 of 3. Write the first part of app.py: imports, page setup, sidebar, loading,
as functions (sidebar(), sources(), the cached loaders). Part 3 adds the pages and main().
- st.set_page_config(page_title="AutoDash", layout="wide"); st.markdown(CSS, unsafe_allow_html=True).
- _optional(name): import a view (or "engine.slides") with importlib inside try/except and
  return the module, or None. A missing file is skipped quietly; a file that exists but fails
  to import shows st.warning("<name> did not load: <error>"), so a broken step is visible.
- Sidebar, top to bottom: title "AutoDash"; segmented("Mode", ["Dashboard", "Explore",
  "Combine"], "mode"); ONE upload section: st.file_uploader("Data files",
  accept_multiple_files=True) with, right under it, a muted line "Maximum file size: 10 GB per
  file." computed from st.get_option("server.maxUploadSize") (MB; 1024 -> "1 GB", 10240 ->
  "10 GB"), then bigfile_view.sidebar_box() when present; "Or try a sample" (a selectbox of
  SAMPLES, None first); a "Read settings" expander (separator, encoding, header row ->
  ReadOptions); Cleaning: segmented(["Clean", "Types only", "Off"]) and a "Dates are day
  first" toggle; last, a "Storage" expander with bigfile_view.storage_settings() when present.
- Sources (uikit.Source): each upload as Source(upload.file_id, upload.name,
  data=upload.getvalue()); the sample as Source("sample:" + key, file_name, data=bytes); each
  table in st.session_state.get("combined", {}) as Source("combined:" + name, name, df=df);
  and bigfile_view.opened() when it returns one. With several, a "File" selectbox picks one.
- st.cache_data functions take small keys and hide big values from hashing with a leading
  underscore: _load(key, name, options, _data) -> list of (table name, DataFrame, notes);
  _clean(key, table, options, _df) -> CleanResult; _profile(key, table, options, _df) -> Profile.
  A Source with df skips _load: one table, named after it.
- In Dashboard and Explore modes: if bigfile_view is present and bigfile_view.wants(src), call
  bigfile_view.render(src) and stop. A file over 1 GB without it: show "Files over 1 GB need
  step 10 (DuckDB)." and stop.
Stop after the loading code. Part 3 continues the same file.
```

```text
STEP 7, PART 3 of 3. Continue app.py (no imports again): the pages and main().
Dashboard mode, for the chosen table:
- Header: the file name; "3,000 rows and 12 columns after cleaning, covering 01 Jan 2023 to
  20 Dec 2024"; a Data health box with health_before and health_after as bars.
- Filters: a date range on the primary date and up to 4 category multiselects. Filtering never
  changes the cached cleaned data.
- Tabs: Dashboard, Insights, Cleaning report, Columns, Data; then Transactions when
  transactions_view is present (render_transactions(df, prof)) and Ask when ask_view is
  present (render_ask(df, prof)).
  Dashboard (uid = source key + "|" + table name):
  - With dashboard_view: tiles, kpi_keys, _ = current_layout(uid, df, prof); kpis =
    build_kpis(df, prof, kpi_keys); a "Customise" toggle (key uid + "_edit"); when it is on,
    customise_panel(uid, tiles, kpi_keys, df, prof, kpis); then draw_kpis(kpis, edit, uid,
    tiles, kpi_keys) and draw_tiles(df, prof, tiles, edit, uid).
  - Without it: build_kpis(df, prof, default_kpi_keys(df, prof)) as a row of st.metric
    (delta_text as the delta), then plan_tiles(df, prof) in cards. Two "half" tiles share a
    row (st.columns(2)); a lone half tile and "full" tiles take the whole row. Each card:
    title_of(tile) in bold, then st.plotly_chart with a unique key; ChartError shows a muted
    line instead.
  - Under the tiles, an "Export" expander: CSV and Excel (up to 1,000,000 rows) of the
    cleaned table; with engine.slides, PowerPoint, PDF and HTML buttons. Each builds its file
    only when pressed (KPIs, top 6 insights, the tiles' figures), then shows a download button.
  Insights: one card per Insight (title bold, detail under it).
  Cleaning report: the actions as a table (category, title, detail, column, rows) and the warnings.
  Columns: role, kind, missing %, distinct values and an example for each column.
  Data: the first 10,000 filtered rows (a column that mixes numbers and text is shown as text)
  and a CSV download of all of them.
Explore mode: explorer_view.render_explorer(tables, name) if present (tables: every table of
every source read in memory), else a note.
Combine mode: combine_view.render_combine(sources, tables_of) if present, with the sources
read in memory and tables_of(src) -> dict table name -> cleaned DataFrame; else a note.
No file yet: a start page with each sample as a card with a "Load sample" button, and
bigfile_view.start_page_card() when present.
main() ends with uikit.footer(). End the file with: if __name__ == "__main__": main()
```

Check: `streamlit run app.py`, pick *Retail sales*, and look for 4 or 5 KPI tiles, 6 or more charts, findings, and a cleaning report that lists the duplicates, the Total row and the Mumbia fix. Switch Cleaning to Off: the raw text values come back.

## Feature steps (8 to 14)

Each feature plugs into the app through the optional views listed in the master prompt, so `app.py` from step 7 doesn't need rewriting. Build them in order: step 10 uses the chart tables of step 6, the SQL of step 8 and, when it is there, the dashboard of step 9.

### Step 8: Explore and SQL, with each example's output (two messages)

```text
STEP 8, PART 1. Write engine/sql.py (CONTRACTS).
- quote(name): double quotes around an identifier, inner quotes doubled.
- check_read_only(query): allow one SELECT or WITH statement only; refuse read_csv,
  read_parquet, read_json, read_text, glob, COPY, ATTACH, INSTALL, LOAD, PRAGMA, EXPORT outside
  quotes and comments: SQLError("Only reading queries are allowed."). bigdata.py uses it too.
- run_query(query, tables): check_read_only first. With duckdb: a new duckdb.connect() per
  query, register only the tables the query names, run, QueryResult(df, seconds, "DuckDB").
  Without duckdb: sqlite3 in memory, the named tables written with to_sql (dates as ISO
  text), engine "SQLite". At most 1,000,000 result rows. Database errors become SQLError.
- class TableFacts(df): rows(); groups(col) -> (distinct values, has empty cells); months(col)
  -> (distinct months, first date, last date, has empty dates); quantile(col, q); above(col,
  cut).
- example_queries(df, engine, facts=None): facts defaults to TableFacts(df). Up to 4 Examples
  with the real (quoted) column names. Each has output = what running it gives, in words, with
  the real numbers; rows = the exact number of rows it returns; columns = its column names.
  1 "First 100 rows": SELECT * FROM data LIMIT 100 -> "The first 100 rows of the table (it has
    3,000), with all 12 columns." ("All 42 rows ..." when 100 or fewer).
  2 "Total <m> by <g>": SELECT g, COUNT(*) AS row_count, SUM(m) AS total, AVG(m) AS average
    FROM data GROUP BY g ORDER BY total DESC -> "5 rows: one for each of the 4 values of
    Region and one for the rows where it is empty. Columns: Region, row_count, and the total
    and average of Sales. The largest total comes first." Without a measure: "Rows per <g>"
    with COUNT(*) only.
  3 "Per month of <d>": date_trunc('month', d) on DuckDB, strftime('%Y-%m', d) on SQLite ->
    "24 rows, one for each month from Jan 2023 to Dec 2024: month and the total of Sales in it.
    The earliest month comes first."
  4 "Largest <m>": WHERE m > <90th percentile, rounded> ORDER BY m DESC LIMIT 50 -> "The 50 rows
    with the largest Sales, largest first, with all 12 columns. 300 rows have Sales above 24486
    (about the top 10%); the query keeps the first 50."
  The row counts must match the real result: SQL groups empty cells into one more row, and the
  cut in the text and in the query is the same rounded number.
```

```text
STEP 8, PART 2. Write explorer_view.py: render_explorer(tables, name). tables: dict table name
-> function returning the cleaned DataFrame.
- A table picker when there are several. Tabs: Table, Columns, SQL.
- Table: a search box (rows containing the text in any column), up to 3 filters (a range slider
  for numbers, a multiselect for categories, a date range), the first 10,000 matching rows, and
  a CSV download of all matching rows.
- Columns: per column its type, missing %, distinct values, min/median/max for numbers, the 5
  most common values for text, and a small histogram or bar chart.
- SQL: a text area (one key per table) starting with the first example, a "Run query" button
  (run_query(sql, {"data": tables[picked], **tables}): "data" is the table picked, and the
  others can be joined by name),
  then the result (rows, seconds, engine) with a CSV download, or the SQLError message.
  Under the box, an expander "Example queries for this table": for each example its title in
  bold, a "Use this" button (an on_click callback writes its SQL into the text area's
  session_state key), st.code(sql, language="sql"), and under it "Output: <example.output>".
```

Check (each line's two numbers must match):

```text
python -c "from engine.sql import example_queries, run_query; from engine.loader import load_file; from engine.cleaner import clean_dataframe; from engine.samples import get_sample; df=clean_dataframe(load_file(*get_sample('retail'))[0].df).df; [print(e.title, len(run_query(e.sql, {'data': lambda: df}).df), e.rows) for e in example_queries(df, 'DuckDB')]"
```

### Step 9: arrange the dashboard: drag, remove, undo (two messages)

```text
STEP 9, PART 1. Write components/arrange/index.html: a Streamlit custom component in ONE file
with inline CSS and JavaScript (no libraries, no build step, offline). It shows the dashboard
as a board: KPI cards in a row on top, chart cards below in a 2-column grid. A "full" card spans
both columns; a "half" card with no half card next to it spans both too, and its sub-line says
"half width, alone in its row".
Protocol (components v1, without streamlit-component-lib):
- On load post to window.parent: {isStreamlitMessage: true, type: "streamlit:componentReady",
  apiVersion: 1}.
- On a message whose data.type is "streamlit:render", read data.args: kpis [{id, label, value}],
  tiles [{id, title, kind, kindLabel, size}], colors {card, line, ink, muted, accent, bad}, sig.
  Redraw from them, except while a drag is in progress.
- After each redraw post {type: "streamlit:setFrameHeight", height: document.body.scrollHeight}.
- To report a change post {type: "streamlit:setComponentValue", dataType: "json", value:
  {nonce: a new random string, sig: args.sig, action, what, kpis: [ids in order],
  tiles: [[id, size], ...]}}. action: move, resize, remove or remove_kpi; what: the card title.
Use:
- Drag with the pointer (pointerdown on a card, then pointermove and pointerup on the document;
  setPointerCapture on the board root). A faded copy follows the pointer, a dashed empty slot
  shows where the card will land, and the other cards move aside live. KPI cards move among KPI
  cards, charts among charts. On drop, report "move" if the order changed; Escape cancels.
- Chart cards have a width button (half <-> full, "resize") and a remove button (x, "remove");
  KPI cards have a remove button ("remove_kpi").
- Keyboard: cards have tabindex 0; arrows move one place, Home and End to the ends, Delete or
  Backspace removes. After a redraw, focus the card at the same position again.
Look: 8px corners, a grip icon, the title, a muted sub-line (chart type and width), colours
from args.colors, the accent border on hover.
```

```text
STEP 9, PART 2. Write dashboard_view.py.
State: st.session_state[f"{uid}_layout"] = {"edited": bool, "tiles": [tile dicts], "kpis":
[keys]}. While edited is False the dashboard is plan_tiles + default_kpi_keys, rebuilt when
filters change. current_layout(uid, df, prof) -> (tiles as Tile objects, kpi_keys, edited).
- Undo: before every change push a deep copy of the state on st.session_state[f"{uid}_undo"]
  (keep 30); _undo(uid) pops it back. _commit(uid, tiles, kpis) pushes, then stores edited=True.
- remove_kpi(...) and _remove(...): commit without the item; toast "Removed X. Undo puts it back."
- The board: components.declare_component("autodash_arrange", path=<this file's folder>/
  components/arrange), declared once. arrange_pane(uid, tiles, kpi_keys, kpis): send kpis (id =
  position as text, label, value = KPI.display()), tiles (id = position, title, kind, kindLabel,
  size), colors, and sig = first 16 hex of sha1 of the JSON of (tile dicts, kpi keys). When the
  value has a nonce not seen before (remember it in session_state) AND its sig equals the
  current sig: rebuild the lists in the new order (an id used twice or unknown -> ignore),
  commit, toast, st.rerun(). Another sig means the board was out of date: ignore the change.
- customise_panel(uid, tiles, kpi_keys, df, prof, kpis): a bordered box with
  segmented(["Arrange", "Suggestions", "Build a chart", "KPI tiles", "Save or load"]) and an
  Undo button beside it (disabled when there is nothing to undo).
  Arrange: the board. Suggestions: planned tiles not on the dashboard, each with "Add".
  Build a chart: kind, x, y, colour and aggregation, then "Add to dashboard".
  KPI tiles: a multiselect of kpi_options (at most 6), in order.
  Save or load: download the layout as JSON; upload one (tiles whose columns are missing are
  skipped and listed); "Back to the automatic dashboard".
- draw_kpis(kpis, edit, uid, tiles, kpi_keys): the KPI row; with edit on, a "Remove" button
  under each KPI.
- draw_tiles(df, prof, tiles, edit, uid, figure=None): the cards as in app.py, each figure from
  figure(tile) when given (big files), else make_chart(df, prof, tile); with edit on, each card
  has small buttons: up, down, width, remove.
```

Check: run the app, turn on Customise, drag a chart to the top, remove a KPI tile, then press Undo twice. The dashboard below the board must follow each change.

### Step 10: big files with DuckDB, uploads up to 10 GB (four messages)

```text
STEP 10, PART 1 of 4. Write engine/bigfile.py: read files too big for memory with DuckDB.
- available(): duckdb imports. kind_of(name): .csv .tsv .txt (+ .gz) -> "csv"; .parquet ->
  "parquet"; .json .jsonl .ndjson (+ .gz) -> "json"; else None.
- cache_root(): env AUTODASH_CACHE or ~/.autodash/bigfiles.
- json_shape(head: bytes, whole=False) -> "lines" (one object per line), "array" ([{...}, ...]
  however it is spread over lines), "document" (records inside other data, a single record, a
  list of lists) or "" (not JSON). Strip a BOM first. Judge from the first 1 MB; when not
  whole, ignore the last (maybe cut) line. A single line holding a list of records -> "document".
- sniff(path) -> FileInfo(path, name, size, mtime, kind, json_format, bom): "lines" ->
  json_format "newline_delimited"; "array" -> "array"; "document" or "" -> BigFileError with
  what to do (upload it if under 1 GB, or save it as JSON Lines or CSV). bom = the file starts
  with EF BB BF.
- start(path, dayfirst=True) -> Job, run in a thread. The key = sha1 of (absolute path, size,
  mtime, dayfirst, FORMAT_VERSION = 1, JSON_READER = 2 for JSON); a finished copy with that key
  is reused. Steps, each updating job.progress (0..1) and job.message:
  1 free disk: need 0.5 x size + 300 MB (1.5 x size when a copy is needed), else BigFileError.
  2 JSON with a BOM: copy the file without the first 3 bytes (DuckDB loses the first record).
  3 raw copy: COPY (SELECT ... FROM <reader>) TO raw.parquet (FORMAT PARQUET, COMPRESSION ZSTD).
    CSV: read the first 50 lines yourself (through gzip for .gz). The separator is the most
    common of , ; tab |. The header is the first line with the most common count of it (title
    lines above it are skipped); names made unique as the loader does. Reader:
    read_csv(path, auto_detect=false, header=false, skip=<lines up to and including the
    header>, delim=<sep>, quote='"', escape='"', columns={<name>: 'VARCHAR', ...},
    null_padding=true, ignore_errors=true). JSON: read_json(path, format=json_format,
    ignore_errors=true). Parquet: read_parquet(path). STRUCT columns become one column per
    field at every depth ("customer.address.city"; list the fields with DESCRIBE SELECT
    unnest(NULL::<type>)); lists of plain values -> array_to_string(x, ', ').
  4 types: clean_dataframe(50,000 sampled rows, CleanOptions.types_only(dayfirst)) tells each
    column's type (number, date, bool or text). Then COPY typed.parquet with the same text
    fixes in SQL: numbers TRY_CAST after removing currency signs and codes, thousands commas,
    spaces and a trailing % ((1,234) -> -1234); dates try_strptime(x, ['%Y-%m-%d',
    '%Y-%m-%d %H:%M:%S', '%Y-%m-%dT%H:%M:%S', '%d/%m/%Y', '%d-%m-%Y', '%d.%m.%Y', '%d %b %Y',
    '%m/%d/%Y']), with '%m/%d/%Y' before '%d/%m/%Y' when dayfirst is False; booleans by their
    two values. notes: one line per changed column, like "Amount: text to number; 12 cells
    could not be read and are empty."
  5 done: job.result = Prepared(key, parquet path, rows, columns [{name, type}], notes, file
    info), also written as JSON next to the copy. job.stop() stops between steps and
    interrupts the connection.
```

```text
STEP 10, PART 2 of 4. Write engine/bigdata.py: class BigData(prepared), queries over the copy.
- rel(where=""): "(SELECT * FROM read_parquet('<path>') <where>)". rows, columns (from prepared).
- query(sql) -> QueryResult: sql.check_read_only(sql) first, then run it with a view named data
  on the copy (a new connection each time).
- sample(n=200_000, where=""): SELECT * FROM rel(where) USING SAMPLE reservoir(n ROWS)
  REPEATABLE (42), kept on the object for each where. head(n=1000, offset=0).
- profile(): profile_dataframe on sample() (roles), with n_rows set to the true row count.
- aggregate(prof, tile, where=""): the same table, with the same columns, as charts.aggregate,
  computed in SQL over every row (trend, bar, parts, rate, heatmap); histogram, scatter and
  table use sample(where=where).
- big_kpis(prof, keys, where=""): the same KPIs as planner.build_kpis, computed in SQL. Give each
  branch its own variable names: one reused name broke the KPI row once.
- facts(): an object with TableFacts' methods, its numbers from SQL over every row (for
  example_queries).
```

```text
STEP 10, PART 3 of 4. Write bigfile_view.py, first part: deciding, storing and opening.
- wants(src) is True for: a file opened by path (src.path set); an upload already stored and
  released (look src.key up first: its data is now empty); an upload of 100 MB or more
  (Parquet: 25 MB) that DuckDB reads as is (CSV/TSV/TXT, Parquet, JSON whose json_shape of the
  first 1 MB is "lines" or "array"); any upload over 1 GB. env AUTODASH_FAST_UPLOAD_MB changes
  the 100 ("off" turns it off). It is False for a table made in Combine (src.df set) and for an upload marked
  to read in memory (below).
- store_upload(name, data): write to cache_root()/uploads/<sha1 of the data, 16 chars>/<name
  without folders or : * ? " < > |>; write "<file>.part" then os.replace; reuse when the file
  exists with the same size. Check free disk first (1.5 x size + 300 MB).
- After storing an upload over 1 GB, free the memory Streamlit holds for it with exactly this:
    def _release(file_id):
        try:
            try:
                from streamlit.runtime.scriptrunner_utils.script_run_context import get_script_run_ctx
            except ImportError:
                from streamlit.runtime.scriptrunner import get_script_run_ctx
            from streamlit.runtime.uploaded_file_manager import UploadedFileRec
            ctx = get_script_run_ctx()
            mgr = ctx.uploaded_file_mgr
            rec = mgr.get_files(session_id=ctx.session_id, file_ids=[file_id])[0]
            mgr.add_file(ctx.session_id, UploadedFileRec(file_id=rec.file_id, name=rec.name,
                         type=rec.type, data=b""))
            return True
        except Exception:
            return False
  Remember released uploads in st.session_state (key -> stored path).
- render(src): an upload over 1 GB that DuckDB can't read (Excel, XML, other JSON) shows
  "<name> (3.2 GB) is too big to read in memory. Save it as CSV or Parquet and upload it
  again." and stops. Otherwise store it (an upload) or take src.path (a file opened by path), then
  start or resume the job; while it runs, a
  progress card (step names, %, a Stop button) inside st.fragment(run_every=1.0). Under the
  header of an upload: "Read with DuckDB because the file is large. The converted copy is kept,
  so it opens in seconds next time." and a "Read in memory instead" button (only up to 1 GB).
  That button, or a failed job for an upload of 1 GB or less (say why in a toast), marks
  src.key to read in memory and calls st.rerun().
- sidebar_box(): a bordered container keyed "ad_path_box": a path text box, an Open button and
  the files opened before. Opening is allowed only for a viewer on the same computer
  (st.context.headers Host is localhost or 127.0.0.1) or inside folders in
  AUTODASH_DATA_FOLDERS. Unless st.session_state["ad_path_shown"] is True, add this CSS, which
  shows the box only while the upload box shows an error (a file over the limit):
  section[data-testid="stSidebar"]:not(:has([data-testid="stFileChipIconError"])):not(:has(
  [data-testid="stFileUploaderFileErrorMessage"])) .st-key-ad_path_box {display: none;}
- opened() -> Source("path:" + path, file name, path=path) for the file opened last, or None.
- start_page_card(): a card "Very large files": uploads up to the server.maxUploadSize limit,
  bigger files by path; an "Open by path" button sets ad_path_shown.
- storage_settings(): disk used by the copies and kept uploads, and a "Delete the copies"
  button that skips files open now.
Stop here. Part 4 continues the same file.
```

```text
STEP 10, PART 4 of 4. Continue bigfile_view.py (no imports again): the pages for a big file.
bd = BigData(job.result); sample = bd.sample(); prof = bd.profile(); uid = src.key + "|data".
The same header, filters and tabs as app.py, with the filters as one SQL where clause (text
values in single quotes, inner quotes doubled) and every number from BigData:
- Dashboard: with dashboard_view, tiles, kpi_keys, _ = current_layout(uid, sample, prof);
  kpis = bd.big_kpis(prof, kpi_keys, where); the Customise toggle, customise_panel and
  draw_kpis as in app.py; draw_tiles(sample, prof, tiles, edit, uid, figure=lambda t:
  charts.draw(bd.aggregate(prof, t, where), prof, t)). Without it: default_kpi_keys and
  plan_tiles, drawn the same way.
- Insights: generate_insights(sample, prof), with the line "Findings use a random sample of
  200,000 rows; totals use every row."
- Cleaning report: job.result.notes. Data: pages of 1,000 rows (bd.head(1000, offset)).
- SQL: a text area on "data" run with bd.query, with example_queries(sample, "DuckDB",
  bd.facts()) under it as in the explorer.
Transactions and Ask are offered only for tables read in memory.
```

Check the engine first. It should print `done`, then 9000 rows, then the count and total of Amount (an error on SUM means Amount was not turned into a number):

```text
python -c "import time; from engine.samples import get_sample; from engine.bigfile import start; from engine.bigdata import BigData; open('big_check.csv', 'wb').write(get_sample('transfers')[1]); j = start('big_check.csv'); [time.sleep(0.5) for _ in range(600) if j.state == 'running']; print(j.state, j.message); b = BigData(j.result); print(b.rows, b.columns[:3]); print(b.query('SELECT COUNT(*) AS n, SUM(Amount) AS total FROM data').df)"
```

Then in the app: upload a CSV of 150 MB or more. The page says it was read with DuckDB; upload the same file again and it should open in a few seconds. Then drop a JSON file shaped `{"data": [...]}`: it must open in memory, not through DuckDB.

### Step 11: Combine: join and stack (two messages)

```text
STEP 11, PART 1. Write engine/join.py (CONTRACTS). class JoinError(ValueError).
- suggest_keys(left, right): column pairs that could join the tables, best first, score 0-1
  from name similarity (difflib, ignoring case, spaces, _ and -) and value overlap (share of
  left's distinct values found in right, compared as trimmed lower-case text; leading zeros
  ignored for digit-only codes). The top 5 with score 0.3 or more.
- join_tables(left, right, left_on, right_on, how): left_on and right_on are one column name
  each (a str, never a list); how is left, inner, outer or right; compare
  keys the same way (helper key columns, dropped after). Shared column names get " (right)" on
  the right copy. Estimate the result size from the key counts first; over 5,000,000 rows ->
  JoinError with the estimate. notes: rows matched, left rows with no match, and "Each left row
  may appear more than once" when right keys repeat.
- stack_tables(frames, add_source): tables under each other, columns matched by trimmed,
  case-insensitive names; missing columns stay empty; a "Source" column with the table name.
  notes list columns only some tables have.
```

```text
STEP 11, PART 2. Write combine_view.py: render_combine(sources, tables_of), where tables_of(src)
returns a dict table name -> cleaned DataFrame. Two tabs:
- Join: pick the left and right tables (from all files), see suggested keys (pick one or choose
  columns), pick how, then a preview of 200 rows with the notes. "Use this table" stores the
  result in st.session_state["combined"][<name>] so the sidebar lists it as a file.
- Stack: pick 2 or more tables (those with the most shared columns are preselected), preview,
  "Use this table".
```

Check (Region should be the first suggestion; the join keeps the number of rows and adds one column; stacking doubles the rows and adds a Source column):

```text
python -c "from engine.join import suggest_keys, join_tables, stack_tables; from engine.loader import load_file; from engine.cleaner import clean_dataframe; from engine.samples import get_sample; df = clean_dataframe(load_file(*get_sample('retail'))[0].df).df; r = df.groupby('Region', as_index=False)['Sales'].sum().rename(columns={'Sales': 'Region total'}); print(suggest_keys(df, r)[:2]); j, n = join_tables(df, r, 'Region', 'Region'); print(df.shape, j.shape, n); s, n2 = stack_tables({'a': df, 'b': df}); print(s.shape, n2)"
```

### Step 12: the Transactions pack (two messages)

```text
STEP 12, PART 1. Write engine/txn.py: review patterns in payments or transfers. They are
places to look, not accusations: each finding states what was seen, with counts and amounts.
- guess_roles(df, prof) -> Roles(date, amount, sender, receiver), from names (date, value date;
  amount, value; sender, from, debtor, payer, originator; receiver, to, creditor, payee,
  beneficiary) and roles. None where unsure.
- findings(df, roles, threshold=10000) -> list of Finding(title, detail, rows), rows being the
  transfers behind it:
  1 just under a threshold: senders with 3+ payments in [0.9 x threshold, threshold) that make
    half or more of what they send;
  2 round amounts: senders with 10+ payments, 80%+ of them multiples of 1,000;
  3 pass-through: accounts with 5+ payments in that send on 80%+ of what they receive within
    2 days;
  4 circles: A -> B -> C -> A within 30 days, among the top 2,000 parties by value;
  5 fan-in / fan-out: 20+ distinct senders to one receiver in 7 days, or the reverse;
  6 bursts: a party whose busiest week is 5x its median week or more;
  7 dormant then active: 90+ days quiet, then 5+ payments.
- party_summary(df, roles, party) -> dict with sent, received, payments_out, payments_in,
  first, last, counterparties (DataFrame: party, sent, received, payments) and daily
  (DataFrame: date, sent, received). Vectorised pandas: a million rows in under a minute.
```

```text
STEP 12, PART 2. Write transactions_view.py: render_transactions(df, prof).
- Role pickers prefilled by guess_roles, and a threshold box (10,000).
- "Review patterns": a note "These are patterns to review. Each has ordinary explanations.",
  then one expander per Finding: detail, its rows as a table and a CSV download.
- "One party": search and pick (top 500 by value): KPIs (in, out, counterparties), money in and
  out per day as bars, top counterparties, its rows.
- "Network": the top 50 parties by value as a plotly network (nodes on a circle, line width by
  amount); hovering shows totals.
```

Check the engine (the four planted patterns should be among the findings: the circle, the payments just under 10,000, the pass-through account and the sender of round amounts):

```text
python -c "from engine.loader import load_file; from engine.cleaner import clean_dataframe; from engine.profiler import profile_dataframe; from engine.txn import guess_roles, findings; from engine.samples import get_sample; df = clean_dataframe(load_file(*get_sample('transfers'))[0].df).df; r = guess_roles(df, profile_dataframe(df)); print(r); [print(f.title, '|', len(f.rows), 'rows') for f in findings(df, r)]"
```

Then in the app: open the *Transfers* sample. The Transactions tab should show the same findings, and One party and Network should work for the circle's accounts.

### Step 13: exports

```text
STEP 13. Write engine/slides.py (CONTRACTS). figures = list of (title, plotly Figure).
- deck_pptx: python-pptx, 16:9. Slide 1: title, date. Slide 2: KPIs as big numbers, 4 a row.
  Slide 3: the top 6 findings. Then a slide per figure, drawn as a NATIVE PowerPoint chart (bar,
  line or pie from the traces' x, y, labels, values) so it stays editable; a figure that doesn't
  map (heatmap, table, scatter) becomes a table of its first 15 rows.
- report_pdf: matplotlib with PdfPages, A4 landscape, the same pages; charts redrawn with
  matplotlib from the traces' data (no kaleido, no internet).
- dashboard_html(title, kpis, insights, figures): one self-contained HTML file; plotly.js is
  included once (include_plotlyjs=True on the first figure, False after).
- Missing python-pptx or matplotlib: ImportError with a plain message naming the package.
```

Check (then open the three files it writes):

```text
python -c "from engine.loader import load_file; from engine.cleaner import clean_dataframe; from engine.profiler import profile_dataframe; from engine.layout import plan_tiles, title_of; from engine.planner import build_kpis, default_kpi_keys; from engine.charts import make_chart; from engine.insights import generate_insights; from engine.slides import deck_pptx, report_pdf, dashboard_html; from engine.samples import get_sample; df = clean_dataframe(load_file(*get_sample('retail'))[0].df).df; p = profile_dataframe(df); k = build_kpis(df, p, default_kpi_keys(df, p)); i = generate_insights(df, p); f = [(title_of(t), make_chart(df, p, t)) for t in plan_tiles(df, p)]; open('check.pptx', 'wb').write(deck_pptx('Retail', k, i, f)); open('check.pdf', 'wb').write(report_pdf('Retail', k, i, f)); open('check.html', 'wb').write(dashboard_html('Retail', k, i, f)); print('open check.pptx, check.pdf and check.html')"
```

### Step 14: Ask a question, with or without a model (three messages)

```text
STEP 14, PART 1. Write engine/llm.py (CONTRACTS) with urllib only. class LLMError(Exception).
from_env(): AUTODASH_LLM_URL, AUTODASH_LLM_MODEL, AUTODASH_LLM_KEY, AUTODASH_LLM_STYLE
(openai | anthropic); None when the URL is unset.
chat(cfg, system, messages): openai style: POST {model, messages (system first), temperature:
0} to the full URL with "Authorization: Bearer <key>". anthropic style: POST {model, system,
messages, max_tokens: 1024} with x-api-key and anthropic-version: 2023-06-01. Return the answer
text. 60 s timeout; the usual HTTPS_PROXY settings; errors -> LLMError with a short reason.
values_allowed(): only when AUTODASH_LLM_VALUES=on are cell values sent; otherwise only column
names, types and counts.
```

```text
STEP 14, PART 2. Write engine/ask.py: answer(question, df, prof, cfg=None) -> Answer(text,
table, figure, sql).
Without a model (always available): rules find the metric (total, average, count, share, top,
trend), the measure and category by column names or their words (case and plurals ignored), a
period ("in 2024", "last month", "Q3", "since March"), filters ("for West", "where Category is
Furniture") and "top 5". Build SQL on "data", run it with run_query, and answer in one
sentence with the number ("Total Sales in 2024 was Rs 12.3M, 41% of all years."), plus the table
and a chart.
With a model: send the question and the schema (names, roles, types, row count; values only if
values_allowed()), ask for one DuckDB query on "data" in a sql code block, run it through
run_query (read-only), send an error back once to be fixed, then word the answer from the
result computed here, never from the model's numbers. Unreachable model: use the rules and say so.
```

```text
STEP 14, PART 3. Write ask_view.py: render_ask(df, prof). A box "Ask a question about this
table", 6 example questions built from the real columns as buttons, then the answer: sentence,
chart, table, and an expander "How this was worked out" with the SQL. One line says whether a
model is connected and what is sent to it.
```

Check without a model (each question should get one sentence with a number):

```text
python -c "from engine.loader import load_file; from engine.cleaner import clean_dataframe; from engine.profiler import profile_dataframe; from engine.ask import answer; from engine.samples import get_sample; df = clean_dataframe(load_file(*get_sample('retail'))[0].df).df; p = profile_dataframe(df); [print(q, '->', answer(q, df, p).text) for q in ('total sales by region', 'average discount in 2024', 'top 5 cities by profit')]"
```

Ask can use your Qwen too. Serve it through an OpenAI-compatible server (Ollama, LM Studio and llama.cpp's server all have one), then set these in the same Command Prompt before `streamlit run app.py`:

```text
set AUTODASH_LLM_URL=http://localhost:11434/v1/chat/completions
set AUTODASH_LLM_MODEL=<the model name your server lists, e.g. qwen3:8b>
```

That address is Ollama's; LM Studio uses port 1234 and llama.cpp's server port 8080. Without these, Ask answers with its rules alone. Cell values are sent only after `set AUTODASH_LLM_VALUES=on`.

## Fix-it prompts

Send these in the chat where the file was written, so the model still has it in view; in a new chat, paste the master prompt first. Replace each `<...>` with your own text.

### A check fails with an error

```text
FIX. The check for step <N> fails with this error:
<paste the whole traceback, from "Traceback" to the last line>
Here is <file path> as it is now:
<paste the file>
Find the cause and fix it. Change only what the fix needs, and keep every name and argument
in CONTRACTS. Reply with the complete corrected file, starting with "File: <path>".
```

For a long file, paste only the function the error is in (the last traceback line that names one of your files gives its name), and end with: `Reply with the complete corrected function only, with the same name, arguments and return value.` Then swap that function into your file.

### The check runs but the result is wrong

```text
FIX. The check for step <N> runs, but the result is wrong.
Expected: <what the step's check says, e.g. about 3,000 rows and Sales as numbers>
Got: <paste the output>
Here is <file path>:
<paste the file>
First say in two sentences what causes it, then give the complete corrected file.
```

### Two files disagree on a name or its arguments

It shows as `AttributeError: module ... has no attribute`, `ImportError: cannot import name`, or a `TypeError` about an unexpected keyword argument or a missing argument.

```text
FIX. <file A> calls <the call, e.g. draw_tiles(df, prof, tiles, edit, uid)>, but <file B>
defines <its def line>. CONTRACTS says: <paste the contract line>.
Change only the file that differs from CONTRACTS. Reply with that complete file.
```

### An error on screen in the app

Find your Streamlit version with `python -c "import streamlit; print(streamlit.__version__)"`.

```text
FIX. The app shows this error when I <what you clicked or chose>:
<paste the red box, or the traceback printed in the terminal>
Streamlit <version>. Here is <file path>:
<paste the file>
Remember: arguments newer than Streamlit 1.40 go through uikit.call; widget keys are unique
(table name + item id); to change a widget's value, use an on_click callback, never set its
session_state key after the widget is drawn. Reply with the complete corrected file.
```

### The answer broke a rule

```text
FIX. Your answer breaks rule <number>: <what it did, e.g. it imports seaborn, or it says
"# rest of the code unchanged", or it loads a font from the internet>. Write the complete
file again without that.
```

### A file too long for one answer

When "continue" keeps failing:

```text
SPLIT. <file path> is too long for one answer. Write it in parts, in the order the step
lists its functions. Part 1 now: the imports and the first third of the functions. End with
the line "# END OF PART 1" and stop. I will ask for the next part.
```

Then send `Part 2, starting after "# END OF PART 1". No imports again. End with "# END OF PART 2".` and so on. Join the parts in order and delete the END lines.

### Continue a file in a new chat

```text
CONTINUE. Here is the first part of <file path>, already saved:
<paste it>
Write the next part as the step below says. Don't repeat code from the first part and
don't import again.
<paste the step's next part>
```

### Review a file before running it

```text
REVIEW. Here is <file path>:
<paste the file>
Compare it with CONTRACTS and the step <N> prompt. List only real problems: a missing or
renamed function, wrong arguments, a name used but never defined or imported, something the
step asks for that is missing. Then give the complete fixed file, or say "No problems".
```

### Tests for a step (needs pytest)

```text
TESTS. Write tests/test_<module>.py with pytest for <file path> (CONTRACTS, step <N>). Use
the samples from engine.samples and pytest's tmp_path; no other files. 6 to 10 short tests:
the normal case, an empty table, a column with every value missing, text with odd spacing
and case, and the numbers the step's check expects. Each test checks a result, not only that
nothing failed.
```

Run them with `python -m pytest -q tests`.

### Too slow

```text
SPEED. <function> in <file path> takes <N> seconds on <rows> rows. Make it faster without
changing what it returns: vectorised pandas (no loops over rows, no apply with a Python
function), each column converted once, nothing computed twice. Reply with the complete
function.
```

If the same fix fails twice, start a new chat with the master prompt, the file and the error; a fresh chat often gets it right first time. Keep a copy of every file that passes its check (a `good` folder) so you can always go back.

## Pitfalls

These went wrong while AutoDash was first built, and small models tend to repeat them. The prompts above already guard against each one, so keep those lines if you shorten a prompt. When you see the symptom, this is the likely cause.

- **One name, two kinds of value.** A count and a label shared one variable name in different branches, and the KPI row crashed with `TypeError: '>' not supported between instances of 'int' and 'str'`. Guard: rule 10, steps 5 and 10.
- **JSON comes in three shapes.** One record per line, one big array, or records inside other data (`{"data": [...]}`). Reading one shape as another gives empty or wrong rows *with no error*, and a byte-order mark at the start makes DuckDB drop the first record. Guard: the loader in step 2 and `json_shape` in step 10.
- **Uploads need two to three times their size in memory.** While a file uploads, Streamlit holds it in memory: a 4 GB file needs 8 to 12 GB of free RAM, and pandas would need several times more. Guard: rule 7, and step 10 stores big uploads on disk and frees that memory. On a computer with little RAM, open very big files by path instead.
- **Streamlit versions differ.** An argument that 1.40 lacks fails with `TypeError: ... got an unexpected keyword argument`. Models also write old names: use `st.rerun` (not `st.experimental_rerun`), `st.cache_data` (not `st.cache`) and `st.fragment` (not `st.experimental_fragment`). Guard: `uikit.call`.
- **Codes lose their leading zeros.** Employee IDs, account numbers, PIN codes and phone numbers turned into numbers become `123` instead of `00123`, and joins stop matching. Guard: step 3 keeps them as text; step 11 compares keys as text.
- **Day and month swapped.** 05/03/2024 is 5 March in India, but pandas reads it as 3 May unless told `dayfirst=True`. Guard: `CleanOptions(dayfirst=True)`, the "Dates are day first" toggle and the format order in step 10.
- **SQL that reaches the disk.** DuckDB can read any file on the computer (`read_csv('C:/...')`) and write one with `COPY`. Guard: `check_read_only` in step 8, also used for big files.
- **One bad chart blanks the page.** An error in one tile stops the whole script. Guard: rule 4; each tile and section has its own try/except.
- **Slow reruns.** `st.cache_data` hashes its arguments on every rerun, and hashing a 1 GB file or table takes seconds each time. Guard: small keys as arguments and the big value behind a leading underscore (step 7, part 2).
- **Widget key errors.** Two widgets with one key fail with a duplicate key error; setting a widget's value after it is drawn fails with "cannot be modified after the widget ... is instantiated". Guard: rule 11, and on\_click callbacks for buttons like "Use this" and "Load sample".
- **Numbers and text mixed in one column.** `st.dataframe` fails with `ArrowTypeError` or `ArrowInvalid`. Guard: step 7's Data tab shows such columns as text; ask for the same wherever a view shows a table.
- **Example SQL that misdescribes its result.** `GROUP BY Region` returns one more row when some Region cells are empty, so the description has to count it. Guard: step 8's check compares each example's `rows` with the real result.
- **A drag applied to a newer layout.** The board can send a change made on a layout that has since changed, which would move the wrong chart. Guard: the `sig` check in step 9.
- **Invented functions.** Small models write arguments that don't exist, or call helpers they never wrote. Guard: rule 9, the REVIEW prompt and running every check.

To have the model check a file against all of these at once:

```text
CHECK PITFALLS. Check <file path> below against this list, then give the complete fixed
file, or say "No problems".
- a variable name reused for a different kind of value in one function
- a file over 1 GB read into pandas
- a Streamlit argument newer than 1.40 not passed through uikit.call
- codes with leading zeros turned into numbers; dates read month first
- SQL run without check_read_only
- a chart or section without its own try/except
- st.cache_data with a big DataFrame or bytes argument whose name doesn't start with _
- widget keys that can repeat, or a widget's session_state set after it is drawn
<paste the file>
```
