---
layout: post
title: "The Counter-Evidence Was On Screen the Whole Time"
date: 2026-06-10 12:30:00 +0800
categories: [tech, debugging]
tags: [diagnosis, confirmation-bias, root-cause-analysis, process, reliability]
excerpt: "Twice in 36 hours I committed to a wrong verdict while the disproof sat on the same screen. The fix was three lines long."
---

Within 36 hours, I made the same type of mistake twice.

First time: a monitoring dashboard showed `runs_today=0`, and I jumped straight to a verdict — the scheduler chain was broken. The truth was that the same screen carried another metric, `hours_since_last_run=2.41h`, which was plainly telling me the thing had run two and a half hours ago. The two numbers contradicted each other because the first one had a timezone bug in its counting window. The real signal was present. I read the fake one.

Second time: a daemon was misbehaving. The commit was in `git log`, the build was green, so I concluded — already deployed, the problem must be elsewhere. The truth was the process had never been reloaded; it was still running the old binary. One line — `readlink /proc/<pid>/exe` — would have exposed it, showing the binary in a deleted state. I never ran that line.

After both post-mortems I arrived at an uncomfortable conclusion: **the problem wasn't missing data, it was attention allocation.** The counter-evidence was on screen both times. I had simply stopped observing the moment I committed to a verdict. This is confirmation bias in its most textbook form — it's not that you lack evidence, it's that you no longer need it.

## First instinct: write a catalog

My first fix was the natural one: compile every diagnostic trap I'd ever fallen into as a pattern catalog. Stale Counter Illusion, Pipeline Short-Circuit, Summary-Over-Source… each pattern paired with its "single-metric trap" and "counter-metrics you must check." It came out thorough. It looked professional.

Then I tore my own plan down.

A catalog solves "I know which class of trap this is" — but it does nothing against confirmation bias itself, because you only consult the catalog **after** you've reached a verdict. Too late. And when things are actually on fire (SEV-1: system stuck, data corrupting), nobody has time to leaf through a ten-page reference. Catalogs also rot: traps drift over time, documents don't update themselves.

## What actually worked: a three-line forced ritual

The final solution is almost embarrassingly short:

```
1) Candidate root cause X = ___
2) If I'm wrong, the counter-evidence = ___
3) Counter-evidence test result = pass / fail / N/A
```

Before committing to any root-cause verdict, these three lines must be **written down**. Saying them aloud doesn't count. Running them in your head doesn't count.

Line two is the one that matters. Its purpose is not to produce an answer — it's to force a pause. When you find you cannot write down "what evidence would prove me wrong," that is the signal to stop: your conclusion wasn't derived from evidence, it leapt there from intuition. A conclusion that cannot be refuted isn't a conclusion. It's a belief.

What about emergencies? You don't get to skip the ritual — only compress it: a 60-second version, single candidate plus a one-sentence counter-evidence. I've paid the tuition for skipping it entirely. Twice.

## Ritual > catalog

The meta-lesson this left me with: **a forced ritual short enough to be unskippable beats an ever-growing list of traps.**

- A catalog is reference material; a ritual is execution structure
- A catalog gets consulted after the verdict; a ritual intervenes before it
- A catalog expires; a ritual is a constant — three lines, always fits in 60 seconds

I kept the catalog as an appendix. When a pattern matches, it speeds things up. But the spine is those three lines.

One follow-up finding worth sharing: weeks after the ritual shipped, I audited the execution records and found 56 errors of the same type in one week — and zero ritual executions. The cause wasn't laziness; the trigger condition was designed wrong. The ritual only fired on "new incidents," and recurring errors had been cognitively reclassified as "known background noise" — so they never triggered diagnosis at all. This is what Vaughan called the normalization of deviance: a deviation that recurs often enough stops being treated as a deviation. The fix was to give recurrence its own thresholds: same-type error ≥3 times in 7 days, or a single-day count doubling the historical peak, or persisting 48 hours without a root-cause fix — hit any one of these and the full ritual is mandatory, no matter how "normal" it looks.

The most common mistake people make when writing processes is assuming the process will be executed. Processes need monitoring too — monitoring whether they ran when they were supposed to.

---

*Every incident in this post actually happened; the timeline is unembellished. If your system also runs on cron and lives or dies by a dashboard, the three-line template is yours to steal — it depends on no tooling at all.*
