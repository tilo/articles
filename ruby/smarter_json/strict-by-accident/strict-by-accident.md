---
title: 'Strict by Accident: Your JSON Parser Isn''t Broken — It''s Answering the Wrong Question'
published: true
description: 'JSON parsers reject the whole document over one stray comma — not because anyone chose strictness, but because they followed the textbook. Why JSON parsing ended up strict by accident, what developers actually complain about, and what a JSON tool built for today''s messy input would do.'
tags: 'ruby, json, parsing, programming'
cover_image: 'https://raw.githubusercontent.com/tilo/articles/main/ruby/smarter_json/strict-by-accident/images/smarter_json_1.1.1_image_3.png'
slug: strict-by-accident
id: 3875897
date: '2026-06-11T17:42:08Z'
---

I was importing a JSON file and it blew up. Not a subtle bug — the parser refused the whole thing. The cause was a single extra comma. After deleting it manually, the import worked.

The comma wasn't the problem. The problem was the manual fix, and the all-or-nothing of it: 99% good data thrown out over one byte. So I went looking to see if it was just me.

It wasn't. The same problem shows up again and again — in JavaScript, Python, Ruby, Go, and more. The shape is always the same: the data is there, and the parser throws all of it away because one byte is off-spec.

## What developers keep hitting

**Almost-JSON from language models.** This is the loudest complaint of the last two years. You ask a model for JSON and get something almost right: wrapped in a ```` ```json ```` fence, followed by "Let me know if you need anything else!", a trailing comma, a number quoted as a string. One developer stated the reliability problem directly: *"It works 95% of the time. The other 5%, you get markdown-wrapped JSON, missing fields, extra commentary… your downstream code crashes."* Prompting harder doesn't fix a probabilistic producer; it moves the failures around.

**One tiny edit, total failure.** The comma above is the common case. One developer removed 154 trailing commas by hand to get a formatter to run — *"I had a lot more important stuff to do."* Another report traces an hour-long webhook outage to a single comma on line 147 of a 200-line config. The data was fine; the byte wasn't.

**NDJSON / JSON Lines.** Logs, exports, event streams, and ML datasets are usually newline-delimited JSON — one object per line. `JSON.parse(File.read("events.jsonl"))` fails on the second line, because the file isn't one JSON value. So you split and loop by hand. Uploading a `.jsonl` to OpenAI's batch API, a developer got *"This line is not parseable as valid JSON"* — for a valid file. The cause was a UTF-8 byte-order mark at the start of the file. The error named the wrong thing; the real cause was an invisible byte.

**Comments.** JSON has no comments, so config files can't be annotated. The common workaround is to either strip comments with a regex before parsing, or to use a configuration option to allow comments. Risky move - what if you get it wrong? The workaround might be worse than the gap it fills.

**Precision, duplicate keys, and dialects.** High-precision numbers — a 64-bit ID, a financial decimal — get silently rounded to a float. Duplicate keys resolve differently across parsers, which is a documented source of security bugs. And "JSON" is not one format but several near-dialects — JSON5, JSONC, NDJSON, whatever your last vendor exported — and you often can't tell which one a `.json` file is until it fails.

When the standard tools won't bend, people build their own. In 2016 a developer named Pēteris was handed a few gigabytes of JSON, most of it fine but scattered with lines a strict parser rejects — a number written `45.`, a `nan` in place of `null`, a stray quote. His parser refused all of it, so he wrote his own tolerant parser by hand to read his file. I bet he would have wanted to spend the time on something more productive.

All these different problems have the same shape: the data was there, and the tool discarded it because it wasn't exactly to spec.

## The ask is small

The striking thing across years of these reports is how modest the request is. Back in 2015 one developer summed up the whole genre:

> *"Comments, trailing commas, and 64-bit integers. That's all I want. Is that really so much? :("*

That was 2015. The same request now covers everything new in how data arrives: read an NDJSON log or export without splitting it by hand; pull the JSON out of an LLM's reply, fences and prose included; take HJSON-style data with its comments and unquoted keys.

That's the whole list: read what's there, keep the data, report findings. Not a new format — a reader that takes a superset of JSON and processes it robustly.

## A different starting point

SmarterCSV taught me one principle: hand back all the usable data, so the user is successful — avoid pre- or post-processing. Rejecting a whole file over one comma is diametrically the opposite. Today's JSON parsers are just that — *parsers*: algorithms designed to police correctness, not maximize usability or user satisfaction.

## Why isn't every parser built that way?

The answer is in the word: it's a *parser*. A parser answers one question — does this input conform to the grammar, yes or no? That is **recognition**. It's the right question when you're checking that a machine produced spec-clean output. It's the wrong question when you have no control over who generates the input and you want the data out of it. That second job is **extraction**.

Recognition and extraction pull in opposite directions. A recognizer's correct answer to "there's a comma on line 147" is a *no* — reject the document. An extractor's correct answer is: *"here's your data; I ignored a comma on line 147."* Same input, opposite result. Most of the time you want the second one.

Nobody decided JSON parsing should be this strict. It's a side effect of how parsers are built historically.

---

> ### Sidebar — How JSON parsing got strict
>
> JSON was built around 2001 to move data between machines. Strictness was correct then: the producer was a program, so a stray comma meant a bug — you'd want the parser to stop and point at it. Strict parsing assumed the producer was careful.
>
> Two decades changed the producer: a vendor exporting its own JSON dialect, a human editing a config, an LLM emitting almost-JSON. The premise no longer holds.
>
> There's a mechanical reason strict parsers are so universal. You write the grammar in BNF, check its class — JSON's is **LL(1)**, one token of lookahead, the standard "simple enough to hand-write" case — open the Compiler Construction Book, and follow the recipe: a tokenizer feeding a parser that walks the grammar. The parser accepts exactly what fits the grammar and rejects everything else. Strictness isn't a decision; it's the result of using the standard algorithm. Leniency is the part you'd have to add deliberately.
>
> The deeper mismatch is older than JSON. That parsing algorithm was meant for programming languages, not for user data — for source code, strictness is correct: a misplaced comma in your code is a bug, and the compiler should stop. JSON parsers inherited that machinery, but user data isn't source code. A stray comma in a data file isn't a bug to reject - it's noise to read past. Same tools, opposite requirements.
>
> Leniency can be designed in — the web already did it. HTML parsers are lenient by specification: HTML5 defines, step by step, how to handle broken markup instead of discarding it. Browsers have done this for two decades, on far messier input than a stray comma. As one developer put it: *"per specifications, json parsing is not lenient, html parsing is lenient."* **Leniency-by-design isn't a hack; it ships in every browser.**
>
> JSON parsers inherited the strict-by-default template and never revisited it. That's how an ecosystem ended up strict by accident.

---

## Stop making people pick a mode

Some parsers offer a way to bend: a flag for trailing commas, another for `NaN`, a dialect setting. That's the wrong model, because in production you can't inspect every payload and decide which mode to enable, and an avoidable production failure is the last thing you want. The input comes from somewhere you don't control and can't predict. A tool that asks you to declare the input's shape up front has handed the impossible part back to you.

And the flags don't get you there. Ruby's standard `json` library has grown more tolerant, but hand it a JSON5 or NDJSON file and it fails even with every leniency option turned on — because unquoted keys and single quotes have no flag at all. You can't configure your way to "read what I was handed."

The fix isn't more flags. It's a different default: one set of safe rules that adapts to whatever arrives, with no mode to pick. Strict JSON is then just the narrowest case of a larger superset the reader already accepts.

There's an architecture difference underneath. The usual way to add leniency is to bolt it on the front: a repair or comment-stripping pass over the text, then a strict parser on the cleaned string. Two passes, an intermediate string, and you still inherit the strict parser's behavior — including lossy numbers. The other way is to build leniency into the parse: one pass, straight to typed data, lossless by default, with a record of what it fixed.

## Repair tools treat the symptom

There is a category of "JSON repair" libraries for JavaScript, Python, PHP, Ruby, ... to clean broken JSON before a strict parser will accept it. They're useful, but look at the shape: a pre-processing pass in front of a strict parser. The most aggressive ones go further and invent data — completing a truncated document with guessed values so it parses. And either you run that pre-processor on every input, which is expensive, or you do it as part of the error handling.

That's treating the symptom. The strip-comments regex, the repair pass, the dialect flag — same move: keep the strict parser, bolt something on the front so reality can get through. That this category exists is the clearest evidence that the strict-only model doesn't fit how JSON is produced and consumed today. The fix isn't pre-processing in front of the parser; it's a reader that doesn't need pre-processing, and that reliably returns only the data that's actually there.

## What a tool for this job looks like

> "Tools should not stand in the way of the user. A data-ingestion library should enable you, not interrupt your work."

I built one: [SmarterJSON](https://github.com/tilo/smarter_json), a JSON processor that focuses on data extraction, not policing of standards. It reads the messy-JSON superset with no modes and no flags — comments, trailing commas, unquoted keys, single quotes, NDJSON in one call, markdown-fenced LLM output — and returns typed, lossless data in one pass, reporting anything it fixed. High-precision numbers stay exact, not rounded to float. It will not invent data: truncated input fails, because the honest answer to "what was the rest?" is unknown. It doesn't validate your schema — that's a separate layer. (In fairness: Ruby's `json` is good and getting better; the difference is the whole superset with zero configuration, not any single feature.)

Here's the honest scorecard against the complaints above:

| The pain                                                          | SmarterJSON                                           |
| ----------------------------------------------------------------- | ----------------------------------------------------- |
| Too strict for hand-written input (no comments / trailing commas) | Addressed                                             |
| One syntax error throws away the whole document                   | Addressed — reads the rest, reports what it skipped   |
| LLM JSON is almost-valid (fences, prose, trailing commas)         | Addressed — wrapper recovery                          |
| JSON buried in prose / markdown must be found first               | Addressed — extracts the payload                      |
| NDJSON / concatenated docs rejected at the 2nd document           | Addressed — one call returns every document           |
| High-precision numbers silently rounded to Float                  | Addressed — BigDecimal by default, Float opt-in       |
| Duplicate keys = undefined behavior (and a security footgun)      | Addressed — explicit policy, every duplicate reported |
| Strict parsers are the wrong default for input you don't control  | Addressed — the core idea                             |
| Weak diagnostics — "it fixed something but I don't know what"     | Addressed — line/column + a warning per fix           |
| A single huge document exhausts memory                            | Partial — multi-doc streams; single huge doc post-1.0 |
| Hand-editing JSON is painful                                      | Addressed — comments, unquoted keys, implicit root    |
| Parses fine but wrong type (`"42"` vs `42`)                       | Not — by design — that's a schema's job               |
| Truncated / cut-off JSON                                          | Not — by design — won't invent missing data           |
| Every tool reinvents JSON handling with its own bugs              | Addressed for Ruby (not other languages)              |
| No Date / extended types (MongoDB `ObjectId`, …)                  | Partial — big ints safe; dates are a schema's job     |
| Dialect fragmentation (JSON5 / JSONC / HJSON / …)                 | Addressed — one superset, no dialect to pick          |
| Leniency assumed to cost speed                                    | Addressed — lenient and fast                          |
| Security at the boundary (parser differentials)                   | Partial — deterministic and observable, not "safer"   |
| One bad payload = production incident                             | Addressed — survive a bad payload                     |

Fourteen addressed, three partial, two it won't do. The two "won'ts" are the point: a tool that never invents data is more trustworthy than one that guesses.

The takeaway isn't that JSON parsers are bad. They're good at the job they were built for — recognizing source-code-grade input from a careful producer. That producer is mostly gone. A developer handed messy JSON should get their data and move on, the same way nobody hand-writes CSV parsing anymore.

## Further reading

- **LLM-generated JSON.** *Structured outputs from LLMs* — the "works 95% of the time, your downstream code crashes" problem: <https://www.aiwisdom.dev/articles/prompt-engineering/structured-outputs-from-llms>. OpenAI batch API, a valid `.jsonl` rejected over an invisible BOM: <https://community.openai.com/t/error-after-creating-batch-job-this-line-is-not-parseable-as-valid-json/728379>
- **Strictness and the all-or-nothing failure.** Nicolas Seriot, *Parsing JSON Is a Minefield* (2016) — how much real parsers disagree: <https://seriot.ch/projects/parsing_json.html>. Pēteris Ņikiforovs, *Parsing malformed JSON* (2016) — the hand-rolled parser: <https://peteris.rocks/blog/parsing-malformed-json/>
- **Comments in config.** Why JSON comments aren't allowed, and what people do instead: <https://jsoneditoronline.org/indepth/parse/json-comments/>
- **Numbers and duplicate keys.** Bishop Fox, *JSON Interoperability Vulnerabilities* (2021) — duplicate keys as a real security footgun: <https://bishopfox.com/blog/json-interoperability-vulnerabilities>
- **Dialect fragmentation.** *Should we consolidate the human-readable JSON efforts?* — json5 issue #190: <https://github.com/json5/json5/issues/190>
- **Leniency by design.** "per specifications, json parsing is not lenient, html parsing is lenient" (Hacker News): <https://news.ycombinator.com/item?id=35475222>
- **Ruby specifics.** Jean Boussier (byroot), *Optimizing Ruby's JSON* (2024) — why "stdlib json is slow" is outdated: <https://byroot.github.io/ruby/json/2024/12/15/optimizing-ruby-json-part-1.html>

