---
layout: post
title: "The Ratio Was Upside Down: Pre-Registration Doesn't Protect Your Definitions"
date: 2026-09-24 04:00:00 +0800
categories: [engineering, methodology]
tags: [pre-registration, metrics, experiment-design, verification, benchmarking, definitions, decision-making]
excerpt: "I pre-registered a performance comparison twice, in two documents, weeks apart. Both said 'pass if the ratio is at least 1.2.' One meant new/old. The other meant old/new. Pre-registration froze my threshold perfectly — and did nothing about the fact that the number it guarded pointed in two opposite directions."
lang: en
---

I like pre-registration. Before running a benchmark or an A/B comparison, I write down what I'm going to measure, what threshold counts as a pass, and what I'll do if it fails. Then I run the thing. The point is simple: it stops me from looking at the result first and deciding afterwards what "good" meant.

Last month it worked exactly as designed, and it still almost let me ship the wrong conclusion. The threshold was frozen. The metric name was frozen. What wasn't frozen was which way up the fraction went.

## What happened

I was comparing an old implementation of a lookup path against a rewrite. I had two planning documents that touched the same comparison:

- A **design note**, written when the rewrite was proposed. It said: *speedup ratio ≥ 1.2 → adopt the rewrite.*
- An **evaluation plan**, written a few weeks later when I actually scheduled the benchmark. It also said: *ratio ≥ 1.2 → pass.*

Same word, same number, same comparison. I read them as the same rule.

They weren't. In the design note, "speedup ratio" was defined further down as `old_latency / new_latency` — bigger means the new code is faster. In the evaluation plan, the ratio was computed in the benchmark script as `new_throughput / old_throughput`... except the script actually divided latencies, so it came out as `new_latency / old_latency` — bigger means the new code is *slower*.

The benchmark came back with a ratio of about 1.25. By one document, that's a clear win. By the other, it's a 25% regression. Both documents were dated, both were written before the data existed, and both would have passed any "did you pre-register this?" audit.

## Why pre-registration didn't catch it

Pre-registration defends against one specific failure: moving the goalposts after you see the result. It pins down *the threshold* and *the decision rule*. It assumes the quantity being thresholded is well-defined.

A ratio is only well-defined once you've fixed three things:

1. **The numerator and denominator** — which system is on top.
2. **The underlying quantity** — latency (lower is better) or throughput (higher is better).
3. **The direction of "good"** — whether the verdict fires when the ratio goes up or down.

Change any one of those and the same threshold means the opposite thing. Change two of them and you're back where you started, which is exactly how I ended up with a script whose name said throughput, whose arithmetic used latency, and whose output happened to agree in format with a document that meant the reverse.

The uncomfortable part is that the *verdict is determined by direction alone*. The number 1.25 carries no information about success until you know which side of 1.0 is "better." And that piece of information was not in either of my threshold lines. It was in a definition, and definitions are where people stop reading carefully because they feel like boilerplate.

## The broader pattern

Once I saw it, I started finding the same shape elsewhere:

- **"Error rate improved by 2x"** — did it halve or double? People say "2x improvement" for both.
- **Percent change vs. percentage points** — "a 5% drop in failures" when the failure rate went from 10% to 5% (a 50% relative drop, 5 points absolute).
- **Latency percentiles** — a lower p99 is better, a higher "requests under 100ms" share is better, and dashboards love to put them side by side.
- **Cost per request vs. requests per dollar** — reciprocal metrics, both called "efficiency."

Every one of these can be pre-registered with a perfectly honest threshold and still produce a verdict that flips depending on which document you read.

## What I changed

None of the fixes are clever. They're about making the direction impossible to leave implicit.

### 1. Write the formula, not the name

The threshold line now includes the formula inline:

> `pass if old_p50_latency / new_p50_latency >= 1.2` (higher = new is faster)

No "speedup ratio" on its own. If the formula isn't on the same line as the threshold, a later reader — including me — will fill it in from memory.

### 2. State the direction in words, next to the number

"Higher is better" or "lower is better," written out. It feels redundant. It's the only part that actually determines the verdict.

### 3. One canonical definition, referenced everywhere

When two documents need the same metric, the second one links to the first instead of restating it. Restating is where drift happens: each rewrite is a chance to invert something without noticing.

### 4. A sanity case before the real run

Before running the actual benchmark, I feed the verdict logic a fake result where I *know* the answer — for example, making the "new" side deliberately sleep for 50ms. If the pipeline reports a pass, the direction is wrong. This takes two minutes and would have caught my bug immediately.

### 5. Print both the ratio and its reciprocal

The benchmark report now shows both directions side by side — say `old/new = 0.80` and `new/old = 1.25` — plus a one-word verdict. Seeing both makes an inversion visually obvious; seeing only one makes it invisible.

## The takeaway

Pre-registration is a guard against *changing your mind*. It is not a guard against *two versions of you meaning different things by the same word*. When a plan exists in more than one place, the thing to diff isn't the threshold — thresholds are easy to keep in sync. It's the definition underneath the threshold, and specifically its direction.

My rule now: any decision that depends on a ratio has to survive a simple question before the data comes in — *"if this number comes back as 1.25, is that good or bad?"* If two documents give different answers, the pre-registration isn't finished yet, no matter how carefully the threshold was written down.

In my case, once I sorted out which way up the fraction was, the rewrite turned out to be about 25% *slower* — the script's 1.25 was `new/old` latency. I didn't adopt it. The part that stays with me is that for about an hour I'd been fully prepared to ship it as a win, with dated, pre-registered paperwork to back me up.
