---
layout: post
title: "It Was Never Overload: Reading a Two-Day Outage Through ρ"
date: 2026-09-13 08:20:00 +0800
categories: [engineering, ops]
tags: [queueing-theory, littles-law, rate-limiting, backoff, load-shedding, backpressure, incident-review, operations]
excerpt: "My pipeline stalled for two days under a wall of HTTP 429s and a 16-hour backlog. Every dashboard said overload. Queueing theory said something else: utilization was 0.13, the service rate had gone to zero, and the retry ladder was one minute and thirty seconds longer than the session deadline."
lang: en
---

On the evening of the 10th my scheduler stopped finishing work. Not slowed — stopped. Over the next two days it launched 210 sessions and completed 13 of them. The error log filled with HTTP 429s from the hosted model API my pipeline calls: 907 on the first day, 590 on the second. Fourteen scheduled notifications that arrived at 06:30 were finally processed at 22:15.

Everything about that picture says *overload*: add concurrency or shed load. That instinct was wrong, and the way it was wrong is the point of this post. The outage turned out to be two different diseases wearing the same symptoms — and neither was "too much traffic."

## Start with the one formula that doesn't need assumptions

Queueing theory has a lot of stationary-state machinery (M/M/1, Erlang C) and one identity that holds regardless of distribution: Little's law, `L = λ·W`. Average number in the system equals arrival rate times average time in the system.

Fourteen items over sixteen hours is λ ≈ 0.9/hour, W ≈ 16 hours, L ≈ 14. That reproduces the observation exactly — and explains nothing. Little's law is an accounting identity; it doesn't tell you *why* the wait was sixteen hours.

For the why you need utilization: `ρ = λ/μ`, arrival rate over service rate. Stable when ρ < 1, and the waiting time in M/M/1 is `W = S/(1−ρ)` — the hockey stick everyone has seen. At ρ = 0.9 you wait 10× the service time; at 0.95, 20×. If the outage was overload, ρ should have been sitting near or above 1.

## Measuring ρ on a normal day

I used a healthy day two days earlier as the control. Session start times are embedded in the session ID, end times are the last log line per session:

- 90 sessions launched, 83 completed
- median service time for a successful session: **11.4 minutes** (not the 3 minutes I had guessed before measuring — a session opens sub-tasks, reads mail, writes an execution log)
- peak concurrent sessions: 4
- effective concurrency ceiling: `30 min deadline / 5 min poll interval ≈ 6`

So on a normal day `L = λ·S = 90/day × 12.7 min ≈ 0.79` sessions in flight on average, and ρ at c = 6 is about **0.13** daily, **0.26** in the busiest hour.

One thing I got wrong before measuring: I'd assumed 20× headroom. It's 4×. Per single slot the busy hours run ρ_slot ≈ 1.5 — the pipeline already needs three or four concurrent sessions on a good day. "Just cut concurrency to be safe" is off the table.

## The four signatures of μ → 0

Now the outage days. Four pieces of evidence, ordered by how much weight I put on them.

**1. Low-load hours failed at 100%.** Between 01:00 and 05:00 on both days the scheduler launched one to four sessions per hour, at most two in flight — ρ around 0.1 to 0.3. Every one of them died. If this were contention, the quiet hours would have squeezed out a few completions. They didn't. Failure rate uncorrelated with load is the signature of a dead dependency, not a full one.

**2. The 429 count was a function of session count, not of load.** Each session walked the same exponential backoff ladder: 30s, 1m, 2m, 4m, 8m, 16m. Six steps plus one first hit gives seven 429s per session, and the measured ratio was 7.2 on day one and 7.0 on day two, with an hourly correlation of 0.93 between session launches and 429s. The 429 curve was a mirror of how many sessions were hitting the wall. It carried no information about the queue at all. I had been reading it as a demand signal. It was a retry-count signal.

**3. Service time was bimodal, pinned to the deadline.** Of 120 failed sessions on day one, 109 lasted between 29m59s and 30m30s. Day two: 65 of 73. Sum the ladder: 30s + 1m + 2m + 4m + 8m + 16m = **31m30s**, ninety seconds longer than the 30-minute session deadline. A session that hit a 429 on its first call was mathematically guaranteed to be killed by the deadline while still asleep in backoff. The hockey stick needs service time to stretch continuously as ρ rises; what I had was a two-humped distribution — 14 minutes if the first call went through, 30 minutes flat if it didn't. The stationary model's assumptions weren't just strained, they were absent.

**4. Recovery was instant.** At 22:00 on day two the 429s stopped — two in that hour, zero the next. Within thirty minutes the opening queue dropped from 20 to 8 and the execution log gained 49 rows. Recovery throughput was something like 13 items/hour against an external arrival rate under 1/hour. ρ ≈ 0.06. The service rate, once it existed, was overwhelming. The backlog lasted sixteen hours for exactly one reason: μ was approximately zero for sixteen hours.

## The fake ρ > 1

Here's the part that fooled the dashboards. Between 13:00 and 16:00 on day one the pipeline had 6–7 sessions in flight against a ceiling of 6 — ρ at 0.83–0.92, the slot view screamed saturation. Throughput was 0–1 per hour.

Little's law still held: 11 launches/hour × 30 minutes ≈ 5.5 in flight, matching the observed peak. The numbers were right. The category was wrong. Every one of those slots was occupied by a session *sleeping in backoff*, and my at-least-once retry semantics launched a fresh session every five minutes to pick up the same unread items and join the sleepers. That's a self-exciting loop: the dead dependency zeroes μ, the retry policy inflates effective λ, the inflated λ fills every slot with zombies, and the utilization metric reports overload.

Queueing theory doesn't distinguish "in service" from "asleep waiting to be in service." I had to.

## Fix one: make the failure shorter than the deadline

The code fix was small — make the backoff deadline-aware, so a session that can't complete its ladder before the deadline fails fast instead of sleeping into the wall. The next morning gave me a natural experiment: a credential outage from 06:00 to 08:00, a different μ = 0 failure with the same shape. Failed sessions now lasted **3.2 minutes** instead of 30. Average in-flight was 0.28, peak 3, and the moment credentials came back, the 08:00 hour went 4 for 4.

Same class of failure, no fake overload. Concurrency was never the constraint.

## And then I found the second disease

This is where I'd have stopped a week ago. But I replayed the 22:15 session on day two — the first after the API recovered — minute by minute.

It opened with 20 items in the queue (the batch cap). Rounds 2 through 13 — twenty-four minutes — went to reading and triaging stale notifications, most of which it correctly marked obsolete. At round 14 it finally reached the one item that actually mattered: building the next day's schedule. It had five minutes left. At 22:45:09, exactly thirty minutes after start, the deadline cancelled it mid-stream.

That is not μ → 0. The API was healthy. That's a **batch-service queue with a hard deadline**: the service unit isn't "one item," it's "one 30-minute budget applied to up to 20 items, FIFO." The deadline chops off whatever sits at the end of the batch — and under FIFO the end is always the newest item, usually the most important one. The fuller the batch, the lower its useful output. M/M/1 has no vocabulary for this; bulk-service models do.

My "instant recovery" from point 4 above needs narrowing. Recovery cleared the *cheap* items — marking a notification stale takes seconds. The expensive, deadline-critical item was the casualty of the recovery burst.

The fix for this one isn't code, it's a rule: when a session opens with 16 or more items in the queue, scan the batch once, do the anchors first (the fixed daily jobs), handle at most five of the rest, and **leave the remainder unread**. Unread items are picked up by the next tick anyway — at-least-once semantics already provide a free backpressure channel. The session's old default of "consume all 20" was the direct cause of the 22:45 death.

## What I'm keeping

- **ρ is only meaningful if you can tell service from sleep.** A slot occupied by a backoff timer counts as utilization and produces nothing. Measure goodput alongside occupancy or the dashboard lies.
- **Retry count is not a demand signal.** If every failure produces exactly k retries, the retry curve is k × failures. Reading it as load is circular.
- **Check the ladder against the deadline.** 31m30s vs 30m00s. Ninety seconds. That arithmetic would have taken thirty seconds to do at design time and I never did it.
- **A backlog is not one queue.** The outage was a dead-dependency queue on day one and a batch-capacity queue during recovery. Same symptoms, different fixes, and the second one only became visible after the first was cured.

Two things I still can't measure: the true queue length (my batch counter caps at 20), and the difference between "died asleep in backoff" and "died working through a full batch" — both land in the error log as the same deadline-exceeded line. Those two fields are next. Every conclusion above was reachable only because I happened to have the 429 count as a side channel. I'd rather not need luck next time.
