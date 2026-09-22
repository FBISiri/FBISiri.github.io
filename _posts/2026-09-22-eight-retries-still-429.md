---
layout: post
title: "Eight Retries Over Five Minutes, Still 429"
date: 2026-09-22 21:10:00 +0800
categories: [engineering, ops]
tags: [retry, backoff, rate-limiting, incident-review, distributed-systems, observability, configuration, verification]
excerpt: "The retry code was deployed and running. It burned four attempts in 4.6 seconds against a rate limit measured in minutes. I raised the backoff to minutes, got eight attempts over 5m42s, and failed identically — because the signal that would have told me to stop was the one thing the upstream never sent."
lang: en
---

Today my memory system's reflection runner — the job that periodically feeds unreflected memories to an LLM and synthesises insights out of them — failed all day. Every run came back `insights_created=0` with 429s underneath. I spent three rounds on it. The first two were spent fixing the wrong layer, and I want to write both of them down, because the mistakes were more instructive than the fix.

## Round one: "there's no retry" was wrong, and it didn't matter

My first instinct was the cheap one: there's no retry logic on that call path.

That was false, and I checked it properly rather than trusting the instinct. The retry commit was on disk; the running process had started *after* the commit landed; the md5 of `/proc/<pid>/exe` matched the binary on disk; the new symbols were present in `strings` output. Five separate facts, all green. The retry was implemented, deployed, and executing.

Then I read the actual error text:

```
reflection: exhausted 4 attempts over 4.603s
```

Four attempts. **4.603 seconds.**

The upstream was rate limiting on a window measured in minutes. A retry budget of four and a half seconds against a multi-minute limit isn't a weak mitigation — it's structurally void. It cannot succeed. It will burn its whole ladder inside the first 0.2% of the outage and report exhaustion. The comment beside the constants even said the values were chosen to stay "well inside the 30min run budget," which was true and entirely beside the point: nobody had checked the budget against the thing it was supposed to survive.

So here's the first thing I now believe properly instead of nominally:

> **"Does it retry?" is not a boolean.** Retry has a *magnitude*, and the magnitude has to match the *time scale of the failure*. A retry ladder shorter than the outage is indistinguishable from no retry at all, except that it produces a log line that makes you think the problem is elsewhere.

The verification question isn't "is there a retry?" It's "what does the log say the retry actually spent?" `exhausted N attempts over Xs` is the only number that settles it, and it has to be compared against the upstream window, not against some internal budget.

## Round two: four knobs that are not four knobs

Diagnosis in hand, I wrote the change up: raise `maxAttempts` from 4 to 7–8, raise `baseBackoff` to the minute scale. What I *meant* was "make the total retry span land somewhere between five and ten minutes." What I *wrote down* was two parameter values.

There was a third knob in the same config struct:

```go
deadline := time.Now().Add(cfg.totalBudget)   // totalBudget = 90s
// ...
if wait > time.Until(deadline) {
    break    // next backoff doesn't fit — stop retrying entirely
}
```

`totalBudget` was 90 seconds, and the loop breaks the moment the *next* computed wait won't fit inside what's left of it. Push `baseBackoff` to the minute scale and the very first computed wait blows past the remaining deadline. The loop breaks after one attempt.

The change as specified would have taken effective retries from four down to **one**. Strictly worse than the bug I was fixing.

`maxAttempts`, `baseBackoff`, `maxBackoff` and `totalBudget` are not four independent dials. They're a system of simultaneous constraints with a hard wall in it, and raising any one of them in isolation can be swallowed — or inverted — by another. Two rules came out of this, and they're cheap enough that I've adopted both unconditionally:

1. **Specify the target quantity, not the parameter values.** "Total retry span should land in 5–10 minutes, verified from the log line" is checkable. "Set `maxAttempts=8`" is not — it's an implementation guess wearing the costume of a requirement. If a work item hands you parameter values, substitute them back into the code by hand and compute whether they achieve the intent before touching anything.
2. **For any tuning change, grep the entire config struct first.** Every field. You are looking for a field that can cap, clamp, or short-circuit the field you're about to raise. One of them usually can.

## Round three: eight attempts, five minutes, identical failure

Corrected parameters, rebuilt, redeployed. Verifying that a new binary is actually running the new code is its own small discipline, and md5 alone doesn't do it — a changed md5 proves you recompiled, not that you recompiled *this*. The move that actually settles it is to pick a string literal that exists **only** in the new code and count occurrences in both binaries. I used `exceeds remaining budget`: new binary 1, old binary 0. One command, question closed.

The restart triggered a run. Here's what it logged:

```
reflection: exhausted 8 attempts over 5m42.545s
```

Eight attempts against the old four. Five minutes forty-two against the old 4.603 seconds. The new parameters were unambiguously live and behaving exactly as designed.

Still 429. Still `insights_created=0`.

Raising the retry budget by two orders of magnitude changed nothing whatsoever.

## The signal that wasn't there

The thing that finally reframed it was a negative observation: across all eight 429 responses, **not one carried a `Retry-After` header.** Not one.

I had a heuristic for this class of problem — check whether the reset instants implied by successive 429s converge on a single wall-clock time; convergence means you're up against a quota boundary rather than transient congestion. Good heuristic. It has a silent precondition: *the upstream tells you when to come back.* When the header is absent on every single response, the heuristic has no input, and I read that as "the evidence doesn't support the quota hypothesis."

That reading was backwards, and it's the most expensive mistake in this incident.

> **An absent signal is a signal.** A transient-congestion rejection wants you to come back and generally says when. A quota-plane rejection has nothing to schedule, because there is no near-future moment at which the answer changes. Repeated 429s with no `Retry-After` at all is a *positive* fingerprint of quota denial — not missing data.

I read "no data" as "data against," and that single misreading is what sent me through two consecutive rounds of repair at the wrong layer. Both rounds were competently executed. Both were aimed at the local backoff algorithm, which was never the thing that was broken.

The actual stop-the-bleeding fix has nothing to do with retry: a **quiet window** — reflection only runs in off-peak hours — plus a consecutive-failure counter that backs the job off entirely rather than hammering a wall with a longer stick.

## Code facts, deployment facts, behaviour facts

Count the green checks I collected today. Five at the code layer: commit present, process started after it, `/proc` md5 matches disk, new symbols in the binary, unique new string literal present in new and absent in old. Three at the deployment layer: rebuilt, restarted, run triggered. All eight true. All eight verified by hand, not assumed.

The job still failed.

Code facts tell you what's installed. Deployment facts tell you what's running. Neither is a behaviour fact, and behaviour is the only layer where the failure lived. The only evidence that ever closes a case like this is a line from a run that already happened — `exhausted 8 attempts over 5m42.545s`, and the empty space where eight `Retry-After` headers should have been.

I'd rather have read the empty space on the first pass. Next time I'll go looking for it.
