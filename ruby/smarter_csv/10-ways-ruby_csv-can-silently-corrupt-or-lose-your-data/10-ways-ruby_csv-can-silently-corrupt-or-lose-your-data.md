---
title: 10 Ways Ruby's CSV.read Can Silently Corrupt or Lose Your Data
published: false
description: 'Ruby''s built-in CSV library has ten failure modes that produce no exception, no warning, and no indication anything went wrong. Your import runs, your tests pass, and your data is quietly wrong.'
tags: 'ruby, csv, rails, programming'
cover_image: 'https://raw.githubusercontent.com/tilo/articles/main/ruby/smarter_csv/10-ways-ruby_csv-can-silently-corrupt-or-lose-your-data/images/10-ways-ruby_csv-can-corrupt-data.png'
slug: 10-ways-ruby_csv-can-silently-corrupt-or-lose-your-data
id: 3356453
---

When having to parse CSV files, many developers go straight to the Ruby `CSV` library — but it comes at the cost of writing boilerplate post-processing, and there are some dangerous pitfalls you might not be aware of.

Ruby's built-in `CSV` library is for many the go-to — it ships with Ruby and requires no dependencies. But it has failure modes that produce **no exception, no warning, and no indication that anything went wrong**. Your import runs, your tests pass, and your data is quietly wrong.

This article documents ten reproducible ways `CSV.read` (and `CSV.table`) can silently corrupt or lose data, with examples you can run yourself, and how SmarterCSV handles each case.

> **Note on `CSV.table`:** It's a convenience wrapper for `CSV.read` with `headers: true`, `header_converters: :symbol`, and `converters: :numeric`.

---

## At a Glance

| # | Ruby CSV Issue | Failure Mode | SmarterCSV fix | SmarterCSV Details |
|---|-------|-------------|:--------------:|---------|
| 1 | Extra columns silently dropped | Data values beyond header count are discarded — no error | by default ✅ | Default `missing_headers: :auto` auto-generates `:column_N` keys for extra values |
| 2 | Duplicate headers — last wins | `.to_h` keeps only the last value; first silently lost | by default ✅ | Default `duplicate_header_suffix:` → `:score`, `:score2`, `:score3` |
| 3 | Empty headers — `""` key collision | Blank headers become `""` keys (`:"" ` with symbol conversion); multiple blanks collide | by default ✅ | Default `missing_header_prefix:` → `:column_1`, `:column_2` |
| 4 | BOM corrupts first header | `"\xEF\xBB\xBFname"` ≠ `"name"` — first column unreachable by key | by default ✅ | Automatic BOM stripping — always on, no option needed |
| 5 | Whitespace in headers ¹ | `" Age"` ≠ `"Age"` — lookup returns nil | by default ✅ | Default `strip_whitespace: true` strips headers and values |
| 6 | `liberal_parsing` garbles fields | Unmatched quotes produce wrong field boundaries — data looks valid | by default ✅ | `on_bad_row:` (default `:raise`) with opt-in `:skip` / `:collect` for quarantine |
| 7 | nil vs `""` for empty fields | Unquoted empty → `nil`, quoted empty → `""` — inconsistent checks | by default ✅ | Default `remove_empty_values: true` removes both from hashes |
| 8 | Backslash-escaped quotes (MySQL/Unix) | `\"` treated as field-closing quote — crash or garbled data | by default ✅ | Default `quote_escaping: :auto` handles both RFC 4180 and backslash escaping |
| 9 | Missing close quote eats rest of file | One unclosed `"` swallows all subsequent rows into a single field | via option | `field_size_limit:` (opt-in) caps field bytes; `quote_boundary: :standard` (default) |
| 10 | No encoding auto-detection | Non-UTF-8 files crash or produce mojibake | via option | `file_encoding:`, `force_utf8: true` (opt-in), `invalid_byte_sequence:` replacement |

¹ The one case where `CSV.table` does better than `CSV.read`: its `header_converters: :symbol` option includes `.strip`, so whitespace is removed from headers. All other nine issues are identical between `CSV.read` and `CSV.table`.

---

## Why These Failures Are Dangerous

**Every single failure in this list is silent.** No exception, no warning, no log line — your import completes successfully and your data is quietly wrong. That's what makes these issues so dangerous: they don't surface in tests, they don't cause immediate errors, and they're easy to miss during code review.

The root cause is that `CSV.read` is a **tokenizer**, not a data pipeline. It splits bytes into fields and hands them back with no normalization, no validation, and no defensive handling of real-world messiness. Every assumption about what "clean" input looks like is left to the caller.

`CSV.table` fixes exactly one issue out of ten — whitespace in headers — because its `:symbol` converter happens to call `.strip`. Everything else is identical.

These aren't obscure edge cases. Extra columns, trailing commas, BOMs, Windows-1252 encoding, duplicate headers, and blank header cells are all common in CSV files exported from Excel, reporting tools, ERP systems, and legacy data pipelines. If your application accepts user-uploaded CSV files, you will encounter these.

The defensive post-processing code required to handle all ten cases correctly — BOM stripping, whitespace normalization, duplicate header disambiguation, extra column naming, consistent empty value handling, backslash quote escaping, field size limits, encoding detection — is non-trivial to write, test, and maintain. Most applications never bother, because the failures are silent.

> **Ready to switch?** → [Switch from Ruby CSV to SmarterCSV in 5 Minutes](https://dev.to/tilo_sloboda/switch-from-ruby_csv-to-smarter_csv-in-5-minutes)

Read on for a detailed explanation and reproducible example for each issue.


---

## 1. Extra Columns Without Headers — Values Silently Discarded

When a row has more fields than there are headers, `CSV.read` maps every extra field to the `nil` key. If there are multiple extra fields, they all compete for the same `nil` key — **only the last one survives**, the rest are silently discarded.

```
$ cat example1.csv
   First Name  , Last Name , Age
Alice , Smith,  30, VIP, Gold ,
Bob, Jones,  25
```


```ruby
rows = CSV.read('example1.csv', headers: true).map(&:to_h)
rows.first
# => {"   First Name  " => "Alice ", " Last Name " => " Smith", " Age" => "  30", nil => ""}
#                             the values "VIP" and "Gold" are silently lost here  ^^^^^^^^^
```

Alice's row has 6 fields but only 3 headers. The extra fields `"VIP"`, `"Gold"`, and `""` (trailing comma) all land on `nil` — each overwriting the last. No error, no warning.

This is common in real-world exports: tools frequently append audit columns, status flags, or trailing commas that don't correspond to headers.

**`CSV.table` has the same problem.**

**SmarterCSV:** The default `missing_headers: :auto` auto-generates distinct names for extra columns using `missing_header_prefix` (default: `"column_"`). The trailing empty field is dropped by the default `remove_empty_values: true` setting. No data loss.

```ruby
rows = SmarterCSV.process('example1.csv')
rows.first
# => {first_name: "Alice", last_name: "Smith", age: 30, column_1: "VIP", column_2: "Gold"}
```

---

## 2. Duplicate Header Names — First Value Silently Dropped

When two columns share the same header name, `CSV::Row#to_h` keeps only the **last** value. The first is silently dropped.

```
$ cat example2.csv
score,name,score
95,Alice,87
```

```ruby
rows = CSV.read('example2.csv', headers: true).map(&:to_h)
rows.first
# => {"score" => "87", "name" => "Alice"}
#    ^^^ first score (95) silently lost
```

Common with reporting tool exports that repeat a column (e.g., two date columns both labeled `"Date"`).

**`CSV.table` has the same problem.**

**SmarterCSV:** disambiguates duplicate headers by appending a number directly: `:score`, `:score2`, `:score3`.

```ruby
rows = SmarterCSV.process('example2.csv')
rows.first
# => {score: 95, name: "Alice", score2: 87}
```

* The default `duplicate_header_suffix: ""` disambiguates by appending a counter: `:score`, `:score2`, `:score3`.
* Use `duplicate_header_suffix: '_'` to get `:score_2`, `:score_3`.
* Set `duplicate_header_suffix: nil` to raise `DuplicateHeaders` instead.

---

## 3. Empty Header Fields — `""` Key Collision

A CSV file with blank header fields (e.g., `name,,age`) gives those columns an empty string `""` as their key. Multiple blank headers all collide on `""` — same overwrite problem as issue #1.

> Note: this is distinct from issue #1. Issue #1 is about extra *data* fields beyond the header count, which get keyed under `nil`. Issue #3 is about blank cells *in the header row itself*, which get keyed under `""`.

```
$ cat example3.csv
name,,,age
Alice,foo,bar,30
```

```ruby
rows = CSV.read('example3.csv', headers: true).map(&:to_h)
rows.first
# => {"name" => "Alice", "" => "bar", "age" => "30"}
#    ^^^ "foo" silently lost — both blank headers wrote to "" key
```

`CSV.table` converts headers to symbols — blank headers become `:"" ` — same collision, different key:

```ruby
rows = CSV.table('example3.csv').map(&:to_h)
rows.first
# => {name: "Alice", :"" => "bar", age: 30}
#    ^^^ "foo" still silently lost
```

**SmarterCSV:** `missing_header_prefix:` (default `"column_"`) auto-generates names for blank headers: `:column_1`, `:column_2`, etc. No collision, no data loss.

```ruby
rows = SmarterCSV.process('example3.csv')
rows.first
# => {name: "Alice", column_1: "foo", column_2: "bar", age: 30}
```

---

## 4. BOM Corrupts the First Header

Files saved by Excel on Windows often include a UTF-8 BOM (`\xEF\xBB\xBF`) at the start. `CSV.read` doesn't strip it, so the BOM is silently prepended to the first header name.

```
$ cat example4.csv
name,age
Alice,30
```

```
$ hexdump -C example4.csv
00000000  ef bb bf 6e 61 6d 65 2c  61 67 65 0a 41 6c 69 63  |...name,age.Alic|
00000010  65 2c 33 30 0a                                     |e,30.|
```

The `ef bb bf` at offset 0 is the UTF-8 BOM — invisible in `cat` output but silently prepended to the first header by `CSV.read`.

```ruby
rows = CSV.read('example4.csv', headers: true).map(&:to_h)
rows.first.keys.first   # => "\xEF\xBB\xBFname"  ← not "name"

rows.first['name']      # => nil   ← first column unreachable
```

The data is there but every lookup on the first column silently returns `nil`. This is a particularly nasty bug because the BOM is invisible in most terminals and editors — the output looks perfectly correct.

**`CSV.table` has the same problem.**

**SmarterCSV:** automatically detects and strips BOMs at the start of the file. Always on, no option needed.

```ruby
rows = SmarterCSV.process('example4.csv')
rows.first.keys.first   # => :name  ← BOM stripped automatically
rows.first[:name]       # => "Alice"
```

---

## 5. Whitespace in Header Names — Silent `nil` on Lookup

`CSV.read` returns headers exactly as they appear in the file, including leading and trailing whitespace. Code that accesses columns by the expected name silently gets `nil`.

```
$ cat example5.csv
 name , age
Alice,30
```

```ruby
rows = CSV.read('example5.csv', headers: true).map(&:to_h)
rows.first
# => {" name " => "Alice", " age " => "30"}

rows.first['name']   # => nil  ← silent miss; key is " name ", not "name"
rows.first['age']    # => nil
```

**`CSV.table` mitigates this:** the `:symbol` header converter includes `.strip`, so whitespace is removed. This is the one issue where `CSV.table` behaves better than `CSV.read`.

**SmarterCSV:**

```ruby
rows = SmarterCSV.process('example5.csv')
rows.first
# => {name: "Alice", age: 30}
```
The default setting `strip_whitespace: true` strips leading/trailing whitespace from both headers and values.


---

## 6. `liberal_parsing: true` Garbles Field Values

`CSV.read` raises `MalformedCSVError` when it encounters an unmatched quote. `liberal_parsing: true` suppresses the error and returns a row anyway — but with wrong field boundaries. The key danger: **without `liberal_parsing` you at least know something is wrong**; with it, corrupted data is silently returned as valid.

```
$ cat example6.csv
name,note,score
Alice,"unclosed quote,99
Bob,normal,87
```

```ruby
# Without liberal_parsing: you know something is wrong
CSV.read('example6.csv', headers: true)
# => CSV::MalformedCSVError: Unclosed quoted field on line 2

# With liberal_parsing: silent corruption
rows = CSV.read('example6.csv', headers: true, liberal_parsing: true).map(&:to_h)
rows.length     # => 1  (not 2 — Bob's row is gone)
rows[0]
# => {"name" => "Alice", "note" => "unclosed quote,99\nBob,normal,87", "score" => nil}
#    Alice's note field swallowed the rest of the file. Bob vanished. No error raised.
```

The garbled row passes validations, gets inserted into the database, and surfaces as a data quality bug later. You don't know what you don't know.

**`CSV.table` has the same problem.**

**SmarterCSV:** `on_bad_row:` (default `:raise`) with opt-in `:skip` or `:collect` for explicit quarantine. Bad rows are isolated — not silently mangled and returned as good data.

```ruby
good_rows = SmarterCSV.process('example6.csv', on_bad_row: :collect)
SmarterCSV.errors
# => {
#     :bad_row_count => 1,
#          :bad_rows => [
#         {
#                 :csv_line_number => 2,
#                :file_line_number => 2,
#             :file_lines_consumed => 2,
#                     :error_class => SmarterCSV::MalformedCSV,
#                   :error_message => "Unclosed quoted field detected in multiline data",
#                :raw_logical_line => "Alice,\"unclosed quote,99\nBob,normal,87\n"
#         }
#     ]
# }
```

> ⚠️ **Fibers:** `SmarterCSV.errors` relies on `Thread.current`, which is shared across all
> fibers in the same thread. If you process CSV in fibers (`Async`, `Falcon`, manual
> `Fiber` scheduling), use `SmarterCSV::Reader` directly — its errors are scoped to the
> instance and are always correct regardless of fiber context.

Or use `SmarterCSV::Reader` directly when you also need access to headers or other reader state:

```ruby
reader = SmarterCSV::Reader.new('example6.csv', on_bad_row: :collect)
good_rows = reader.process
reader.errors   # same structure as above
```

* `on_bad_row: :raise` (default) fails fast.
* `on_bad_row: :skip` discards bad rows silently — count available via `SmarterCSV.errors[:bad_row_count]`.
* `on_bad_row: :collect` quarantines them — available via `SmarterCSV.errors` or `reader.errors`.
* `on_bad_row: ->(rec) { ... }` calls your lambda per bad row.

---

## 7. `nil` vs `""` for Empty Fields — Inconsistent Empty Checks

`CSV.read` treats unquoted empty fields and quoted empty fields differently:

- Unquoted empty (`,,`) → `nil`
- Quoted empty (`,"",`) → `""`

```
$ cat example7.csv
name,city
Alice,
Bob,""
```

```ruby
rows = CSV.read('example7.csv', headers: true).map(&:to_h)

rows[0]['city']        # => nil   (unquoted empty)
rows[1]['city']        # => ""    (quoted empty)

rows[0]['city'].nil?   # => true
rows[1]['city'].nil?   # => false  ← same semantic meaning, different Ruby type
```

Both rows have no city. But your code sees two different things. Any check using `.nil?`, `.blank?`, `.present?`, or a simple `if row['city']` will behave differently depending on how the upstream exporter happened to quote the empty field. No two exporters agree on this.

**`CSV.table` has the same problem.**

**SmarterCSV:** `remove_empty_values: true` (default) removes both from the hash. With `remove_empty_values: false`, both become `nil`. Consistent either way.

```ruby
# remove_empty_values: true (default) — both empty cities are dropped from the hash
rows = SmarterCSV.process('example7.csv')
rows[0]   # => {name: "Alice"}
rows[1]   # => {name: "Bob"}

# remove_empty_values: false — both normalized to nil
rows = SmarterCSV.process('example7.csv', remove_empty_values: false)
rows[0]   # => {name: "Alice", city: nil}
rows[1]   # => {name: "Bob",   city: nil}
```

---

## 8. Backslash-Escaped Quotes — MySQL / Unix Dump Format

MySQL's `SELECT INTO OUTFILE`, PostgreSQL `COPY TO`, and many Unix data-pipeline tools escape embedded double quotes as `\"` — not as `""` (the RFC 4180 standard). Ruby's `CSV` only understands the RFC 4180 convention, so a backslash before a quote is treated as two separate characters: a literal `\` followed by a `"` that immediately **closes the field**.

```
$ cat example8.csv
name,note
Alice,"She said \"hello\" to everyone"
Bob,"Normal note"
```

**Scenario 1 — crash** (at least you know something went wrong):

```ruby
rows = CSV.read('example8.csv', headers: true)
# => CSV::MalformedCSVError: Illegal quoting in line 2.
```

**Scenario 2 — silent garbling** with `liberal_parsing: true`:

```ruby
rows = CSV.read('example8.csv', headers: true, liberal_parsing: true)
rows[0]['name']   # => "Alice"
rows[0]['note']   # => "She said \\"   ← field closed at the backslash-quote; rest lost
rows[1]['name']   # => "hello"          ← Alice's leftovers eaten as Bob's name
rows[1]['note']   # => nil
```

No exception. No warning. `rows.length` is still 2. The data just quietly moved to the wrong fields.

**`CSV.table` has the same problem** — and adding `liberal_parsing: true` makes it silently worse.

**SmarterCSV:** `quote_escaping: :auto` (default since 1.0) detects and handles both `""` and `\"` escaping row-by-row. No option required.

```ruby
rows = SmarterCSV.process('example8.csv')
rows[0]   # => {name: "Alice", note: "She said \"hello\" to everyone"}
rows[1]   # => {name: "Bob",   note: "Normal note"}
```

---

## 9. Missing Closing Quote Consumes the Rest of the File

A single unclosed `"` anywhere in the file causes the parser to enter quoted-field mode and treat everything that follows — newlines included — as part of one field. **All remaining rows are swallowed into a single field value.**

```
$ cat example8.csv
name,age
"Alice,30
Bob,25
Carol,40
```

```ruby
rows = CSV.read('example8.csv', headers: true)
rows.length         # => 1  (not 3)
rows.first['name']  # => "Alice,30\nBob,25\nCarol,40"
#                         ^^^ entire remainder of file in one field
```

On a large file this is an OOM risk: the parser accumulates an ever-growing string until EOF or memory exhaustion. There is no field size limit, no timeout, and no error until the file ends — by which point you've read the whole file into RAM.

**`CSV.table` has the same problem.**

**SmarterCSV:** `field_size_limit: N` raises `SmarterCSV::FieldSizeLimitExceeded` as soon as any field or accumulating multiline buffer exceeds N bytes — the parse stops immediately. Additionally, `quote_boundary: :standard` (the default since 1.16.0) means mid-field quotes don't toggle quoted mode at all, reducing the attack surface.

```ruby
good_rows = SmarterCSV.process('example8.csv',
  field_size_limit: 10_000,
  on_bad_row: :collect,
)
SmarterCSV.errors
# => {
#     :bad_row_count => 1,
#          :bad_rows => [
#         {
#                 :csv_line_number => 2,
#                :file_line_number => 2,
#             :file_lines_consumed => 3,
#                     :error_class => SmarterCSV::MalformedCSV,
#                   :error_message => "Unclosed quoted field detected in multiline data",
#                :raw_logical_line => "\"Alice,30\nBob,25\nCarol,40\n"
#         }
#     ]
# }
```

> ⚠️ **Fibers:** `SmarterCSV.errors` relies on `Thread.current`, which is shared across all
> fibers in the same thread. If you process CSV in fibers (`Async`, `Falcon`, manual
> `Fiber` scheduling), use `SmarterCSV::Reader` directly — its errors are scoped to the
> instance and are always correct regardless of fiber context.

Or use `SmarterCSV::Reader` directly when you also need access to headers or other reader state:

```ruby
reader = SmarterCSV::Reader.new('example8.csv',
  field_size_limit: 10_000,
  on_bad_row: :collect,
)
good_rows = reader.process
reader.errors   # same structure as above
```

---

## 10. No Encoding Auto-Detection — Crash or Mojibake

`CSV.read` assumes UTF-8. CSV files exported from Excel on Windows are typically Windows-1252 (CP1252), which encodes accented characters (é, ü, ñ) differently from UTF-8.

```
$ cat example9.csv
last_name,first_name
Müller,Hans
```

The file is saved in Windows-1252 encoding — `ü` is stored as `\xFC`, not as UTF-8.

**Scenario 1 — crash** (the better outcome — at least you know):

```ruby
rows = CSV.read('example9.csv', headers: true)
# => Encoding::InvalidByteSequenceError: "\xFC" from ASCII-8BIT to UTF-8
```

**Scenario 2 — silent mojibake** (the worse outcome):

```ruby
# Specifying the wrong encoding suppresses the error
rows = CSV.read('example9.csv', headers: true, encoding: 'binary')
rows.first['last_name']                # => "M\xFCller"  ← garbled string
rows.first['last_name'].valid_encoding? # => true  ← Ruby thinks it's fine!
```

The mojibake string passes `.valid_encoding?`, passes database validations, gets stored, and surfaces as a display bug weeks later in production.

**`CSV.table` has the same problem.**

**SmarterCSV:** `file_encoding:` accepts Ruby's `'external:internal'` transcoding notation; `force_utf8: true` transcodes to UTF-8 automatically; `invalid_byte_sequence:` controls the replacement character for bytes that can't be transcoded.

```ruby
rows = SmarterCSV.process('example9.csv',
  file_encoding: 'windows-1252:utf-8')
rows.first[:last_name]   # => "Müller"
```

---

## The Fix

```ruby
gem 'smarter_csv'
```

```ruby
# Before
rows = CSV.read('data.csv', headers: true).map(&:to_h)

# After
rows = SmarterCSV.process('data.csv')
```

SmarterCSV handles all ten cases with sensible defaults — BOM stripping, whitespace normalization, duplicate header disambiguation, extra column naming, consistent empty value handling, backslash quote escaping, field size limits, and encoding control. No boilerplate, no post-processing pipeline, no silent data loss.

* **GitHub:** [github.com/tilo/smarter_csv](https://github.com/tilo/smarter_csv)
* **Docs:** [Full documentation](https://github.com/tilo/smarter_csv/blob/master/docs/_introduction.md)
* **RubyGems:** [rubygems.org/gems/smarter_csv](https://rubygems.org/gems/smarter_csv)
