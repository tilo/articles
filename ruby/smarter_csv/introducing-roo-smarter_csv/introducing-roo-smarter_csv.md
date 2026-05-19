---
title: Introducing roo-smarter_csv — A Drop-In Roo CSV Backend That's 3–4.6x Faster
published: false
description: 'roo-smarter_csv replaces Roo''s built-in CSV backend with SmarterCSV. Same Roo spreadsheet API, 3–4.6× faster parsing, automatic col_sep / row_sep detection, and robust handling of real-world CSV files.'
tags: 'ruby, csv, roo, rails'
cover_image: 'https://raw.githubusercontent.com/tilo/articles/main/ruby/smarter_csv/introducing-roo-smarter_csv/images/roo-smarter_csv_1.0.0.png'
slug: introducing-roo-smarter_csv
date: '2026-05-18T00:00:00Z'
id: 3696158
---

**Roo** is great at hiding the differences between CSV, XLSX, ODS, and friends behind one spreadsheet-style API, but its CSV processing is slow.

Meet [**roo-smarter_csv**](https://github.com/tilo/roo-smarter_csv) — a drop-in backend that makes Roo's CSV path **3–4.6× faster** and significantly more robust against messy real-world data, without changing a single line of your existing Roo code.

```ruby
gem 'roo-smarter_csv'
```

```ruby
require 'roo-smarter_csv'

spreadsheet = Roo::Spreadsheet.open('people.csv')
spreadsheet.cell(2, 1)   # => "John"
spreadsheet.row(2)       # => ["John", 30, "john@example.com", 50000]
```

That's the whole integration. `Roo::Spreadsheet.open`, `cell`, `row`, `column`, `each`, `parse`, `first_row` / `last_row`, `first_column` / `last_column`, `to_csv` — all the methods you already use keep working exactly as before. Under the hood, [SmarterCSV](https://github.com/tilo/smarter_csv) does the parsing.

---

## Why a new backend?

For CSV specifically, Roo delegates to Ruby's built-in `CSV` library, and there are three long-standing **problems with Ruby `CSV`**:

1. **It's slow.** Ruby's `CSV` is the bottleneck in many real-world Roo CSV pipelines.
2. **It requires manual configuration.** You need to provide the parameters for column and row separators, amongst other things.
3. **It's fragile against real-world data.** Inconsistent separators, BOMs, mixed quote styles, embedded newlines, and numeric coercion edge cases can cause silent data corruption — see [10 Ways Ruby's CSV.read Can Silently Corrupt or Lose Your Data](https://dev.to/tilo_sloboda/10-ways-rubys-csvread-can-silently-corrupt-or-lose-your-data-1g02) for a tour.

`roo-smarter_csv` swaps SmarterCSV in as the parser while keeping Roo's spreadsheet API as the public model. SmarterCSV has been around for **14 years** — a battle-tested library you can rely on. You get its speed, robustness, and automatic detection of parameters — and nothing about how you call Roo changes.

---

## Performance: 3–4.6× faster than Roo::CSV

Speedup measured against Roo's built-in CSV backend, using SmarterCSV 1.17.1:

| File                           | Speedup |
| ------------------------------ | ------: |
| PEOPLE_IMPORT_B.csv            |   2.98× |
| uscities.csv                   |   4.22× |
| uszips.csv                     |   4.45× |
| worldcities.csv                |   **4.58×** |
| embedded_newlines_60k.csv      |   3.84× |
| heavy_quoting_60k.csv          |   3.42× |
| many_empty_fields_60k.csv      |   3.36× |
| sample_100k.csv                |   3.17× |
| sensor_data_50krows_50cols.csv |   3.23× |
| tab_separated_60k.tsv          |   3.14× |
| utf8_multibyte_60k.csv         |   3.17× |

The speedup holds up across the awkward shapes Ruby's CSV tends to choke on — heavy quoting, embedded newlines, many empty fields, UTF-8 multibyte content, tab-separated input. SmarterCSV's C extension does the heavy lifting; the Roo grid model is populated from the parsed rows.

For deeper benchmark background, see the recent [SmarterCSV 1.16 release notes](https://dev.to/tilo_sloboda/smartercsv-116-released-faster-than-csvread-bad-row-quarantine-instrumentation-new-features-improved-api-30km) and the [SmarterCSV 1.15.2 benchmark write-up](https://tilo-sloboda.medium.com/smartercsv-1-15-2-faster-than-raw-csv-arrays-benchmarks-zsv-and-the-full-pipeline-2c12a798032e).

---

## Benefits beyond speed

The performance number is the headline, but the *robustness* improvements are arguably more valuable on a production import path.

### Automatic separator detection

SmarterCSV's `col_sep: :auto` and `row_sep: :auto` are the effective defaults. CSV exports from Excel, MySQL, PostgreSQL, Google Sheets, and assorted European tools use different separators (`,`, `;`, `\t`) and different line endings (`\n`, `\r\n`, `\r`). With Ruby's `CSV` you have to know upfront — guess wrong and you get one giant row, or one column with embedded commas. With `roo-smarter_csv` you usually don't need to specify anything:

```ruby
Roo::Spreadsheet.open('us_export.csv')        # comma-separated, LF
Roo::Spreadsheet.open('eu_export.csv')        # semicolon-separated, CRLF
Roo::Spreadsheet.open('mysql_export.tsv', extension: :csv)   # tab-separated
```

All three Just Work.

### Automatic numeric conversion

`convert_values_to_numeric: true` is on by default. Cells containing `"30"` become `30`, `"1.5"` becomes `1.5`. With Ruby's CSV you'd write the coercion yourself, or pass `converters: :numeric` and pray you don't have ZIP codes with leading zeros (Ruby CSV mangles those - SmarterCSV handles those correctly).

```ruby
spreadsheet.cell(2, 2)   # => 30        (Integer, not "30")
spreadsheet.cell(2, 4)   # => 1.5       (Float, not "1.5")
```

### UTF-8 BOM handling

Excel loves to write a UTF-8 BOM at the start of CSV files. Ruby's CSV will happily put a `<0xfeff>` at the start of your first header. SmarterCSV strips the BOM transparently — your first column header is the actual header.

### Robust quote handling

SmarterCSV 1.16 ships RFC 4180–compliant quote boundary handling by default, plus `quote_escaping: :auto` that handles both `""` (RFC) and `\"` (MySQL, PostgreSQL `COPY TO`) escape conventions row-by-row. Mid-field quotes (`5'10"`, `O'Brien`) no longer toggle quoted mode and silently corrupt rows.

### Same spreadsheet model

Critically, none of this changes how Roo presents the data. SmarterCSV row hashes are an internal parsing representation; Roo still stores everything in its coordinate-based cell grid, so `cell(row, col)`, `row(n)`, `column(n)`, `first_row`, `last_row`, `each`, `parse`, and `to_csv` all behave exactly as Roo users expect.

Blank rows stay addressable, too — `roo-smarter_csv` sets `remove_empty_hashes: false` so Roo's row numbering matches the file even when rows are empty.

---

## Installation

```ruby
# Gemfile
gem 'roo-smarter_csv'
```

```bash
bundle install
```

```ruby
# Anywhere in your app's boot path (config/application.rb, an initializer, etc.)
require 'roo-smarter_csv'
```

`require "roo-smarter_csv"` loads both `roo` and `smarter_csv` and registers `Roo::SmarterCSV` as Roo's CSV handler. From that point on, every `Roo::Spreadsheet.open(...)` on a CSV file routes through SmarterCSV.

---

## Usage examples

### Drop-in replacement

```ruby
require 'roo'
require 'roo-smarter_csv'

csv = Roo::Spreadsheet.open('people.csv')

csv.cell(2, 1)      # => "John"
csv.cell(2, 2)      # => 30
csv.row(2)          # => ["John", 30, "john@example.com", 50000]
csv.first_row       # => 1
csv.last_row        # => 4

csv.each do |row|
  # process row
end
```

### TSV (tab-separated)

```ruby
csv = Roo::Spreadsheet.open(
  'people.tsv',
  extension: :csv,
  csv_options: { col_sep: "\t" }
)
```

### StringIO / in-memory input

```ruby
io = StringIO.new("Name,Age\nAlice,30\nBob,25\n")
csv = Roo::Spreadsheet.open(io, extension: :csv)
csv.row(2)   # => ["Alice", 30]
```

### Passing SmarterCSV options directly

```ruby
csv = Roo::Spreadsheet.open(
  'data.csv',
  smarter_csv: {
    col_sep:    ';',
    quote_char: '"',
    encoding:   'utf-8',
  }
)
```

---

## Options: two namespaces, clear precedence

`roo-smarter_csv` understands two option namespaces and resolves them in a predictable order.

### `smarter_csv:` — the primary namespace

Anything SmarterCSV accepts can go here:

```ruby
Roo::Spreadsheet.open('data.csv',
  smarter_csv: {
    col_sep: ';',
    row_sep: "\n",
    quote_char: '"',
    encoding: 'utf-8',
  })
```

### `csv_options:` — Roo compatibility namespace

If you already pass `csv_options:` to Roo, the following four keys are bridged into the effective SmarterCSV options:

- `col_sep`
- `row_sep`
- `quote_char`
- `encoding`

No other Roo options are treated as CSV parser settings.

### Precedence rules

1. Start with SmarterCSV defaults.
2. Apply `roo-smarter_csv` Roo-compatibility overrides (notably `remove_empty_hashes: false`).
3. Copy the supported keys from `csv_options:` into the effective SmarterCSV options.
4. Apply `smarter_csv:` on top.
5. If the same key exists in both places, `smarter_csv:` wins and a warning is emitted.

```ruby
Roo::Spreadsheet.open(
  'data.csv',
  csv_options:  { col_sep: ';'  },
  smarter_csv:  { col_sep: "\t" }   # ← wins, warning emitted
)
```

This means existing Roo code that passes `csv_options:` keeps working unchanged, and you can opt into the full SmarterCSV option surface whenever you want.

### Effective defaults

When you pass no options at all, the effective configuration is:

- `col_sep: :auto` — auto-detect separator
- `row_sep: :auto` — auto-detect line endings
- `quote_char: '"'`
- `downcase_header: true`
- `strings_as_keys: false`
- `convert_values_to_numeric: true`
- `remove_empty_hashes: false` *(Roo-compat override)*
- `headers_in_file: true`

That covers the vast majority of real-world CSV inputs without any configuration.

---

## What it does *not* change

`roo-smarter_csv` is intentionally narrow in scope:

- **It only affects CSV.** Roo's XLSX, ODS, and other backends are untouched.
- **It preserves Roo's coordinate model.** SmarterCSV's hash-of-symbols rows are an internal parsing artifact — the public API is still spreadsheet-style cells, rows, and columns.
- **It preserves Roo's single-sheet CSV behavior.** A CSV file is still a single sheet.
- **It preserves `to_csv` export** for the in-memory spreadsheet representation.

If you've been using Roo's CSV path and the rest of your code expects Roo's grid API, nothing in your code needs to change.

---

## When to reach for it

`roo-smarter_csv` is the right choice when:

- You already have a Roo-based pipeline and don't want to rewrite it.
- You import CSV files from heterogeneous sources (different tools, locales, separator conventions).
- Your imports are big enough that a 3–4× speedup matters.
- You've been hitting silent data quality bugs caused by Ruby CSV's defaults.

If you're starting fresh and don't need Roo's multi-format abstraction, use SmarterCSV directly — you get the same speed plus a richer hash-based API (chunked processing, instrumentation hooks, bad-row quarantine, `key_mapping`, column selection, and more). See [Switch from Ruby CSV to SmarterCSV in 5 Minutes](https://dev.to/tilo_sloboda/switch-from-ruby-csv-to-smartercsv-in-5-minutes-3636).

---

## Further Reading

- **[10 Ways Ruby's CSV.read Can Silently Corrupt or Lose Your Data](https://dev.to/tilo_sloboda/10-ways-rubys-csvread-can-silently-corrupt-or-lose-your-data-1g02)**
- **[Switch from Ruby CSV to SmarterCSV in 5 Minutes](https://dev.to/tilo_sloboda/switch-from-ruby-csv-to-smartercsv-in-5-minutes-3636)**

---
## That's it ✨

```ruby
gem 'roo-smarter_csv'
```

```ruby
require 'roo-smarter_csv'
```

Two lines, no API change, 3–4.6× faster CSV imports and much better tolerance for real-world data.

* **GitHub:** [github.com/tilo/roo-smarter_csv](https://github.com/tilo/roo-smarter_csv)
* **RubyGems:** [rubygems.org/gems/roo-smarter_csv](https://rubygems.org/gems/roo-smarter_csv)
* **SmarterCSV:** [github.com/tilo/smarter_csv](https://github.com/tilo/smarter_csv)
* **Roo:** [github.com/roo-rb/roo](https://github.com/roo-rb/roo)

Issues, feedback, and PRs welcome.
