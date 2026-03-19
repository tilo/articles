---
title: Your Ruby CSV Import Ran Successfully — Your Data May Still Be Wrong
published: false
description: '10 failure modes in Ruby CSV that produce no exception, no warning, and no indication that anything went wrong. Your import runs. Your tests pass. Your data is quietly wrong.'
tags: 'ruby, csv, rails, DataEngineering'
cover_image: null
slug: your-ruby-csv-import-ran-successfully-your-data-may-still-be-wrong
id: 3370680
---

> Are you sure that Ruby CSV imported all your data — and correctly? 🤔

I wasn't looking for bugs. I was improving the performance of smarter_csv, then added a new round of tests — including some borrowed from Ruby CSV's own test suite as a sanity check. Then I started thinking through error scenarios. What I found was genuinely surprising — and it also led me to realize that smarter_csv needed a reliable mechanism for bad row quarantine.

I found 10 failure modes in Ruby CSV that produce no exception, no warning, and no indication that anything went wrong. Your import runs. Your tests pass. Your data is quietly wrong.

The one that still gets me: CSV's numeric conversion silently converts the ZIP code "00123" into 83 🤯. Not a rounding error — a completely different number, because it interprets leading zeros as octal. ZIP codes, customer IDs, order numbers — all silently replaced with wrong integers that pass every validation, look plausible, and get stored in your database.

Or this: a user uploads a tab-separated file named `.csv` — your file-type guard passes, Ruby CSV sees no commas, treats each entire row as a single field, and returns what looks like valid data. All column structure is silently gone.

I wrote up all 10, with reproducible examples you can download and run yourself:
👉 [10 Ways Ruby's CSV.read Can Silently Corrupt or Lose Your Data](10 Ways Ruby's CSV.read Can Silently Corrupt or Lose Your Data)

If you're ready to switch, SmarterCSV 1.16 is out — now 1.8×–8.6× faster than `CSV.read` end-to-end, with a bad-row quarantine system, and  instrumentation hooks:
👉 [SmarterCSV 1.16 Released](https://dev.to/tilo_sloboda/smartercsv-116-released-faster-than-csvread-bad-row-quarantine-instrumentation-new-features-2a060)

Found something? Issues and feedback always welcome on GitHub and if I don't respond, don't be shy: ping me on LinkedIn. I love improving the code.
