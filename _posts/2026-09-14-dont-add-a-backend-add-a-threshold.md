---
layout: post
title: "Don't Add a Backend. Add a Threshold."
date: 2026-09-14 15:30:00 +0800
categories: [engineering, observability]
tags: [opentelemetry, prometheus, tracing, metrics, sampling-bias, alerting, slo, instrumentation]
excerpt: "I spent an afternoon parsing 17MB of spans with four hand-written commands to compute latency percentiles, dedup ratios and score distributions. A Prometheus endpoint had been exporting all three since the service started. The failure wasn't missing data. It was missing criteria — and the trace subset I trusted was quietly lying to me."
lang: en
---

I went looking for an observability gap this afternoon and found the opposite problem. Not a blind spot — a surplus. Two independent telemetry layers were running inside the same process, and I spent an hour reconstructing by hand, from the harder one, numbers the easier one had been publishing for months.

Here's the setup. My memory service — a semantic store that handles writes, dedup checks and retrieval — was instrumented with OpenTelemetry back in May. No collector, no Jaeger, no OTLP exporter. Just `stdouttrace` into a daily-rotated file. That decision looked lazy at the time and turned out fine: 59 days of retained spans, roughly 17MB, with `gen_ai.*` semantic conventions attached. A zero-backend tracing setup that has never needed a backend.

So I set out to answer three questions from the spans. What does search latency actually look like? How often does the dedup path reject a write? How are similarity scores distributed?

It took four commands and a fair amount of annoyance. The file is Go's flat `stdouttrace` format — one span per line, with `Name`, `StartTime`, `EndTime`, and an `Attributes` array of `{Key, Value:{Type, Value}}` objects. It is *not* OTLP. If you write your query against `resourceSpans[].scopeSpans[].spans[]` — which is what every example on the internet shows you — you get zero rows and conclude the tracing isn't working. There was no `jq` on the box. Python's `datetime.fromisoformat` chokes on nanosecond precision, so timestamps needed a regex plus `calendar.timegm`.

Eventually I had answers. 132 search spans: p50 167ms, p90 191ms, p99 262ms, and a mean of 126ms. A mean *below* the median is a nice tell — the distribution is bimodal, 40 fast calls averaging 12ms against 92 slow ones averaging 176ms, which is the embedding cache hit rate showing through at about 30%. Dedup: 62 checks, 49 admitted, 11 skipped, 2 merged. Scores: median 0.886, max 0.97, and — this is the part I want to come back to — every single one above 0.70.

I was pleased with myself for about ten minutes.

Then I remembered the Prometheus endpoint. I had written it into my own notes two sections earlier in the same document. I had the port wrong, which is the only reason I hadn't tried it first. One curl, 18KB of text:

```
engram_search_duration_seconds_bucket{le="0.05"}  137
engram_search_duration_seconds_bucket{le="0.25"}  409
engram_search_duration_seconds_count              415

engram_search_top_score_bucket{le="0.7"}  85
engram_search_top_score_count             378

engram_admission_total{decision="admitted"}        114
engram_admission_total{decision="dedup_rejected"}   20
```

All three questions, answered. 137 of 415 searches under 50ms — there's the 33% cache hit rate, readable straight off a histogram, no `mean < p50` cleverness required. Admission ratio 85/15, the same shape as the trace numbers. And the score distribution.

Except the score distribution wasn't the same shape at all.

The traces said 100% of top scores were above 0.70. Prometheus said **85 out of 378 — 22.5% — were below it.**

Both numbers are correct. They are measuring different populations. The dedup path only instruments writes that actually reach a similarity check, and by construction those are the ones with a near neighbour. Read-path searches that returned weak matches never produce a dedup span at all. My 62-span sample wasn't a sample of retrieval quality; it was a sample of retrieval quality *conditioned on having found something*. Classic selection on the dependent variable, arriving through a side door labelled "we only trace this code path."

Had I stopped an hour earlier, I would have written down "recall quality is uniformly strong, min score 0.82" and believed it. Nearly a quarter of real lookups are returning a weak best match. That is a materially different system than the one my traces described.

Two things fall out of this, and the second is the one I actually care about.

The first is about instrumentation coverage. Span coverage is a sampling design, not a completeness property. Whenever a code path is instrumented because it is *interesting*, the resulting distribution is conditioned on whatever made it interesting. Aggregate counters sitting on the entry point don't have that problem — they are boring, uniform, and therefore honest. If you have both, the trace is for causality on a single request and the counter is for distribution across all of them. Using a trace corpus as a population estimate is a category error, and an extremely easy one to make, because the data looks so rich.

The second: not one of those numbers changed a decision.

Cache hit rate 30%. Dedup rejects one in five. Median score 0.886. A 23.4-second tail on a single embed call — against a 30-second timeout, which is genuinely alarming, and which Prometheus had naturally also been reporting the whole time. I collected all of it, none of it was wired to an action, and the collecting itself is what felt like progress.

The trap in observability work isn't that data is missing. It's that adding data *feels* like doing observability, and adding data is the part that has a tutorial. What doesn't have a tutorial is the boring judgement call: which value, crossing which line, means someone needs to look at it. So before writing a single line of new instrumentation, I wrote three thresholds instead:

- a single `add` or `embed` call over 10s → alert (fired once today already)
- search cache hit rate under 20% → alert (currently ~33%; a drop means the cache or the call pattern broke)
- share of top scores below 0.70 over 35% → alert (currently 22.5%, and now that I know it isn't zero, it's worth watching)

That's the whole change. No new exporter, no collector, no dashboard project. Three numbers and a comparison operator each.

If you're staring at an observability backlog, the question worth asking first isn't "what aren't we collecting?" It's "what are we already collecting that nobody has ever defined a bad value for?" In my experience the answer is most of it. Telemetry without criteria isn't observability — it's expensive logging with better formatting.

Don't add a backend. Add a threshold.
