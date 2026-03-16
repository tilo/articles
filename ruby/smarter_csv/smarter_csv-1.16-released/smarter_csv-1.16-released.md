---
title: 'SmarterCSV 1.16 Released — Faster Than CSV.read, Bad Row Quarantine, new Featured, and a Improved API'
published: false
description: 'SmarterCSV 1.16 brings major performance gains (up to 129× faster than CSV.table), a new bad-row quarantine system, a significantly expanded API, and 696 new tests.'
tags: 'ruby, csv, performance, rails'
cover_image: null
slug: smarter_csv-1.16-released
id: 3357262
---

SmarterCSV 1.16 is out — it brings major performance gains, a new bad-row quarantine system, a significantly expanded API, and 696 new tests.

```ruby
gem 'smarter_csv', '~> 1.16'
```

---

## Performance: Faster Than CSV.read

The headline number that usually surprises people: SmarterCSV 1.16 returns **fully processed symbol-keyed hashes with numeric conversion** — and still beats `CSV.read` (which returns raw string arrays with no post-processing at all):

| Comparison | Speedup |
|---|---|
| vs `CSV.read` (raw arrays) | **1.7×–8.6× faster** |
| vs `CSV.table` (symbol keys + numeric conversion — the fair comparison) | **7×–129× faster** |
| vs SmarterCSV 1.15.2 | **up to 2.4× faster** |
| vs SmarterCSV 1.14.4 | **9×–65× faster** |

Measured on 19 benchmark files, Apple M1 Pro, Ruby 3.4.7. The 129× figure is on a 117-column import file where `CSV.table`'s overhead compounds with column count.

The `CSV.table` comparison is the apples-to-apples one: both produce symbol-keyed hashes with numeric conversion. That's what you actually need in a Rails app.

### What drove the gains

**C extension (accelerated path):**
- `ParseContext` architecture — all per-file parse options wrapped in a GC-managed `TypedData` object, built once after headers load. Eliminates ~10 `rb_hash_aref` calls per row that previously re-read the options hash on every single row.
- `getbyte` + `byteslice` everywhere — character lookups in the inner loop replaced with `getbyte` (returns Integer directly, ~5–10 ns, zero allocation vs ~30–50 ns for a one-char String).
- `memchr` skip-ahead — jumps to the next quote character inside quoted fields instead of advancing byte-by-byte. This is the biggest win on long-field files (up to 2.4×).
- Column-filter bitmap — `headers: { only: [...] }` uses a precomputed packed bitmap; excluded columns are skipped before any string allocation or hash insertion.

**Ruby path:**
- Direct hash construction — `parse_line_to_hash_ruby` builds the result hash directly from `String#split` for unquoted lines. Eliminates the intermediate `Array` from `parse_csv_line_ruby` and a second full-row iteration.
- Hot-path option caching — `@quote_char`, `@col_sep`, `@field_size_limit`, and seven other options precomputed as instance variables after headers load.

### Column selection speedup

When using `headers: { only: [...] }` to keep a subset of columns, excluded columns are skipped entirely in the C hot path:

| Columns kept | Speedup vs no selection |
|---|---|
| 2 of 500 | ~16× faster |
| 10 of 500 | ~8× faster |
| 50 of 500 | ~3× faster |

```ruby
# Only extract the 3 columns you need from a 500-column file
rows = SmarterCSV.process('wide_export.csv',
  headers: { only: [:id, :name, :email] })
```

---

## Bad Row Quarantine

Real-world CSV files are malformed. Until now, SmarterCSV raised on the first bad row and stopped — all-or-nothing. 1.16 adds a full quarantine system.

### `on_bad_row:`

```ruby
# :raise (default) — fail fast, same as before
SmarterCSV.process('data.csv')

# :skip — continue, count available afterwards
SmarterCSV.process('data.csv', on_bad_row: :skip)
puts SmarterCSV.errors[:bad_row_count]   # => 3

# :collect — continue and keep the error records
good_rows = SmarterCSV.process('data.csv', on_bad_row: :collect)
SmarterCSV.errors[:bad_rows].each do |rec|
  Rails.logger.warn "Bad row #{rec[:csv_line_number]}: #{rec[:error_message]}"
  Rails.logger.warn "Raw: #{rec[:raw_logical_line]}"
end

# callable — inline handling, no Reader instance needed
bad_rows = []
good_rows = SmarterCSV.process('data.csv',
  on_bad_row: ->(rec) { bad_rows << rec })
```

Each error record contains:

```ruby
{
  csv_line_number:     3,
  file_line_number:    3,
  file_lines_consumed: 1,
  error_class:         SmarterCSV::HeaderSizeMismatch,
  error_message:       "extra columns detected ...",
  raw_logical_line:    "Jane,25,Boston,EXTRA_DATA\n",
}
```

### `field_size_limit:` — DoS protection

One unclosed `"` in a large file causes Ruby's `CSV` to read the entire rest of the file into a single field — a silent OOM risk. `field_size_limit: N` raises `FieldSizeLimitExceeded` as soon as any field or accumulating multiline buffer exceeds N bytes:

```ruby
SmarterCSV.process('uploads/user_data.csv',
  field_size_limit: 1_000_000,   # 1 MB per field
  on_bad_row: :collect)
```

### `bad_row_limit:` — abort after too many failures

```ruby
reader = SmarterCSV::Reader.new('data.csv',
  on_bad_row: :collect,
  bad_row_limit: 10)

begin
  result = reader.process
rescue SmarterCSV::TooManyBadRows => e
  puts "Aborting: #{e.message}"
  puts "Collected so far: #{reader.errors[:bad_rows].size}"
end
```

### `SmarterCSV.errors` — class-level error access (1.16.1)

Previously, accessing `bad_row_count` after a class-level call required switching to `SmarterCSV::Reader`. 1.16.1 exposes errors directly:

```ruby
SmarterCSV.process('data.csv', on_bad_row: :skip)
puts SmarterCSV.errors[:bad_row_count]   # => 3

SmarterCSV.process('data.csv', on_bad_row: :collect)
puts SmarterCSV.errors[:bad_rows].size   # => 3
```

`SmarterCSV.errors` is thread-local — each thread in Puma/Sidekiq tracks its own state independently. It stores the result of the most recent call on the current thread.

> ⚠️ **Fibers:** `SmarterCSV.errors` uses `Thread.current`, which is shared across all fibers
> in the same thread. If you process CSV in fibers (`Async`, `Falcon`, manual `Fiber`
> scheduling), use `SmarterCSV::Reader` directly — its errors are scoped to the instance.

---

## Expanded Read API

### `SmarterCSV.parse` — parse strings directly

```ruby
# Before — had to wrap in StringIO
reader = SmarterCSV::Reader.new(StringIO.new(csv_string))

# Now
rows = SmarterCSV.parse(csv_string)
```

Drop-in equivalent of `CSV.parse(str, headers: true, header_converters: :symbol)` — with numeric conversion included.

### `SmarterCSV.each` / `Reader#each` — row-by-row enumerator

`Reader` now includes `Enumerable`:

```ruby
# Lazy pipeline — process a 10M-row file with constant memory
SmarterCSV.each('huge.csv')
  .lazy
  .select { |row| row[:status] == 'active' }
  .first(100)
  .each { |row| MyModel.create!(row) }
```

### `SmarterCSV.each_chunk` / `Reader#each_chunk`

```ruby
SmarterCSV.each_chunk('data.csv', chunk_size: 500).each_with_index do |chunk, i|
  puts "Importing chunk #{i}..."
  MyModel.import(chunk)
end
```

---

## Quote Handling Improvements

### `quote_boundary: :standard` (default — minor breaking change)

Previously, a quote character mid-field (e.g. `5'10"` or `O'Brien`) could toggle quoted mode and silently corrupt the row. The new default `:standard` mode only recognizes quotes as field delimiters at field boundaries — RFC 4180 compliant behavior.

In practice, mid-field quotes were already producing silent corruption in 1.15.x, so this is a bug fix that looks like a breaking change. Use `quote_boundary: :legacy` only if you deliberately relied on the old behavior.

### `quote_escaping: :auto` (default)

MySQL `SELECT INTO OUTFILE`, PostgreSQL `COPY TO`, and many Unix tools escape quotes as `\"` instead of `""` (RFC 4180). `:auto` mode handles both conventions row-by-row without configuration:

```ruby
# Both of these parse correctly with the default quote_escaping: :auto
rows = SmarterCSV.process('mysql_export.csv')    # uses \"
rows = SmarterCSV.process('excel_export.csv')    # uses ""
```

---

## Instrumentation Hooks

```ruby
SmarterCSV.process('large_import.csv',
  chunk_size: 1000,
  on_start: ->(info) {
    puts "Starting import of #{info[:file_size]} bytes"
  },
  on_chunk: ->(info) {
    puts "Chunk #{info[:chunk_number]}: #{info[:total_rows_so_far]} rows so far"
  },
  on_complete: ->(info) {
    puts "Done: #{info[:total_rows]} rows in #{info[:duration].round(2)}s"
    puts "Bad rows: #{info[:bad_rows]}"
  }
)
```

---

## Writer Improvements

### IO and StringIO support

```ruby
# Write to any IO object
SmarterCSV.generate($stdout) do |csv|
  csv << {name: "Alice", age: 30}
end

# Write to a StringIO buffer
buffer = StringIO.new
SmarterCSV.generate(buffer) do |csv|
  csv << {name: "Alice", age: 30}
end
csv_string = buffer.string
```

### Generate to String directly

```ruby
csv_string = SmarterCSV.generate do |csv|
  csv << {name: "Alice", age: 30}
  csv << {name: "Bob",   age: 25}
end
```

### New writer options

```ruby
SmarterCSV.generate('output.csv',
  encoding:          'ISO-8859-1',
  write_nil_value:   'NULL',
  write_empty_value: '',
  write_bom:         true,        # UTF-8 BOM for Excel compatibility
) do |csv|
  csv << row
end
```

### Streaming mode

When `headers:` or `map_headers:` is provided at construction, the Writer skips the internal temp file entirely — the header line is written immediately and each `<<` streams directly to the output. No API change; existing code benefits automatically.

---

## Bug Fixes

**1.16.0:**
- Empty/whitespace-only header cells now auto-generate names (`column_1`, `column_2`, …) instead of colliding on `""` — fixes [#324](https://github.com/tilo/smarter_csv/issues/324) and [#312](https://github.com/tilo/smarter_csv/issues/312)
- Mid-field quotes no longer corrupt unquoted fields (`quote_boundary: :standard`)
- All library output now goes to `$stderr` — nothing written to `$stdout`
- Writer temp file no longer hardcoded to `/tmp` (fixes Windows)

**1.16.1:**
- `col_sep` in quoted headers was parsed incorrectly — fixes [#325](https://github.com/tilo/smarter_csv/issues/325) (thanks to Paho Lurie-Gregg)
- Quoted numeric fields were not converted to numeric

---

## Deprecations

These options still work but emit a warning — update when convenient:

| Old | New |
|-----|-----|
| `only_headers:` | `headers: { only: [...] }` |
| `except_headers:` | `headers: { except: [...] }` |
| `remove_values_matching:` | `nil_values_matching:` |
| `strict: true` | `missing_headers: :raise` |
| `strict: false` | `missing_headers: :auto` |
| `verbose: true` | `verbose: :debug` |
| `verbose: false` | `verbose: :normal` |

---

## By the Numbers

| | |
|---|---|
| RSpec tests | 714 → 1,410 (+696) |
| Line coverage | 100% |
| Benchmark files | 19 |
| New options | 10 |
| New exceptions | 2 |

---

## Further Reading

- **[10 Ways Ruby's CSV.read Can Silently Corrupt or Lose Your Data](https://github.com/tilo/smarter_csv/blob/master/docs/ruby_csv_pitfalls.md)** — the silent failure modes that make switching worthwhile, with reproducible examples for each
- **[Switch from Ruby CSV to SmarterCSV in 5 Minutes](https://github.com/tilo/smarter_csv/blob/master/docs/migrating_from_csv.md)** — a practical migration guide with before/after examples and a quick-reference options table

---

## Links

- **GitHub:** [github.com/tilo/smarter_csv](https://github.com/tilo/smarter_csv)
- **Docs:** [Full documentation](https://github.com/tilo/smarter_csv/blob/master/docs/_introduction.md)
- **RubyGems:** [rubygems.org/gems/smarter_csv](https://rubygems.org/gems/smarter_csv)
- **Full changelog:** [1.16.0](https://github.com/tilo/smarter_csv/blob/master/docs/releases/1.16.0/changes.md) · [1.16.1](https://github.com/tilo/smarter_csv/blob/master/CHANGELOG.md)
- **Benchmarks:** [Full benchmark tables](https://github.com/tilo/smarter_csv/blob/master/docs/releases/1.16.0/benchmarks.md)
