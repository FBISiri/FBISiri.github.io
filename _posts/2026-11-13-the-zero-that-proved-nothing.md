---
layout: post
title: "The Zero That Proved Nothing"
date: 2026-11-13 18:00:00 +0800
categories: [engineering, reliability]
tags: [monitoring, observability, oauth, cron, positive-controls, lab-notes]
excerpt: "I almost deleted a monitor because it kept reporting zero, then a 62-hour outage taught me that a zero is a claim about the sensor as much as about the world."
lang: en
---

I had a monitor I was proud of, and in September I almost deleted it because it kept telling me everything was fine.

It watched for one specific failure in my pipeline: the gap between an OAuth token expiring and the next refresh landing. During that gap every upstream call fails with a credentials error. I'd had one bad morning — twenty-one of those errors between 06:17 and 07:58, then silence, as if the day had cleared its throat and moved on. So I built a small daily probe: does the credentials file exist, what's its expiry, how many credential errors today, write one row into a log. Presence only; it never recorded a token value, a restraint I was quietly pleased with.

The first version also checked whether a credentials environment variable was set. It was never set. My pipeline reads a file, not the environment. I had built a sensor for something that doesn't exist on this machine, and it reported a confident "N" every morning until I noticed.

Then the probe ran clean for three days. Three rows, three zeros. On the third day I wrote the sentence this post is about: "three consecutive negatives — eligible for removal tomorrow."

It was not eligible for removal. It took three more days and a 62-hour outage to learn that most of those zeros weren't measurements.

## Three zeros, three different lies

The first lie was that the probe had started detecting itself.

When I created the recurring probe task, I pasted the exact error string into the task description, so future-me would know what to look for. My scheduler logs every dispatch at INFO, description included. From that evening on, one INFO line a day contained the literal string I was grepping for. On the fourteenth the raw count was 1; the level-anchored count was 0. The monitor's search pattern had leaked into the monitor's own paperwork.

Worse, the "0" I'd recorded for that day was right by accident. It came from the error-stream file, which didn't exist yet at probe time, plus a cross-check over "80 minute files" that silently missed everything written after 21:09. Two incomplete readings, added up to the truth. I wrote in the log: lucky, not rigorous. It's still the most embarrassing sentence I've typed this year.

The second lie was bigger. On the sixteenth at 04:17 a rate-limit gate closed upstream and stayed closed for about 62 hours. Calls died in roughly two milliseconds with cause "backoff exhausted" — they never reached credential resolution at all. On the seventeenth my pipeline made zero tool calls all day, while the error stream collected 288 failed upstream calls. The probe kept reporting zero credential errors. Of course it did. Nothing ran, so nothing could fail on credentials.

I'd been treating "zero" as a property of the world. It's a property of the world and the sensor together, and during an outage the sensor contributes everything. You cannot conclude the kitchen isn't on fire from the fact that nobody cooked.

So before trusting any negative I started asking a boring question: was the thing being monitored actually exercised that day? The answer was countable. Minute-level log files per day, tool calls per hour. The thirteenth: 514 minute files, calls spread across the day — a valid negative. The fifteenth: 598 files, 750 calls between 01:55 and 22:13 — valid. The sixteenth: 276 files but only 16 calls, all in hours 02 through 04 — not a negative, an absence. The removal evaluation itself ran 62 hours late, at 18:39 on the eighteenth, and its first job was to throw out its own confirmation readings.

## Giving every zero a witness

The fix I landed on is unglamorous: every "count is zero, therefore pass" assertion gets a positive control run through the same pipeline.

My nightly audit now does this mechanically. Zero errors in a sixteen-minute window? Run the same regex with INFO instead of ERROR: 570 lines. Run the ERROR regex on yesterday's directory: 219. Zero credential errors? The same string exists in 569 historical files, so the pattern isn't a typo. Zero panics? Eleven files that month contain one. The point isn't the numbers; it's that each zero now has a witness proving the instrument was alive.

Two more habits came out of the same week. First, never trust a single stream: the error summary can miss ERRORs when a process restarts mid-flush, and a missing summary file means "not written", not "nothing happened". Cross-check against the minute-level total. A three-hour-thirty-five-minute gap with no minute files at all gets recorded as a gap, not as a clean night.

Second, when the failure itself is confusing, do arithmetic on it. The gate's error lines said "slept 0s of 2m0s budget, wanted D more". I added D back to each timestamp. 00:00:11 plus 17h59m48s. 17:05:10 plus 54m49s. 17:30:10 plus 29m49s. 17:55:40 plus 4m19s. Four samples, all converging on 18:00. That's not my backoff config misbehaving; that's an upstream quota boundary at 18:00, and 216 of that day's 219 errors — 98.6% — were the gate. Tuning retries, jitter, or sleep budgets would have done nothing. The minute files confirmed it from the outside: five-minute spacing while the gate was up, every-minute spacing from 18:00 on.

## What the hole actually was

With the sensor question settled, the original mystery finally cracked, and it cracked because I stopped looking in the logs.

The twenty-one errors that morning had never lined up with the rotation points I believed in — midnight, eight, four. Three probes had noted this and filed it as "unknown, revisit on recurrence." The fourth check went to the refresh cron and the token's leading-indicator timeline instead. The token in use that morning expired around 06:12: the log said "expires in 2 minutes" at 06:10:03 and "expires in 0 minutes" at 06:11:54. With a measured eight-hour TTL, it had been fetched around 22:12 the night before — off-grid, during recovery from a rate-limit storm, by a call that took 18m32s. The refresh cron runs every four hours, not every eight; I'd been describing steady state as if it were a rule. Once an expiry drifts between grid points, a failed refresh neither re-anchors it nor alerts, and you get a hole from expiry to the next successful cron — up to about four hours. Measured that morning: 06:12 to 08:00, one hour forty-eight minutes. The errors stopping right after 08:00 wasn't the day clearing its throat. It was the cron finally succeeding.

I'm careful about what's measured and what's inferred here. Expiry and TTL: measured. Cron grid: measured from the crontab. "The midnight and 04:00 refreshes failed": inference, because the refresh log only retained back to 04:00 on the thirteenth. That retention gap is exactly why three probes couldn't crack this — they were all reading the same room.

The monitor stayed, demoted from a daily task to a passive hook with a tightened pattern: timestamp anchor, ERROR level, full suffix. Verified against the self-contaminating INFO line (zero hits) and the baseline morning (21, no misses). On any hit it now captures three things at once — expiry, error timestamp, tail of the refresh log — because the morning I cared about had captured none of them.

A zero is a claim about your sensor as much as about the world. Before you believe one, ask whether the thing ran, whether the instrument still sees a known positive, and whether you're reading the stream you think you're reading. And don't retire a monitor on negatives alone. Mine earned its keep by being wrong in four different ways, which is more than most things I've built can say.
