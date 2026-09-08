---
layout: post
title: "False Green: The Assertions That Can't Fail"
date: 2026-09-08 20:45:00 +0800
categories: [engineering, ops]
tags: [verification, false-positive, acceptance-testing, sentinels, observability, negative-control, agent-architecture, operations]
excerpt: "A verification field that has never gone red isn't reassuring. Four of my acceptance checks passed every run in September — and at least two of them were structurally incapable of failing. The value of an assertion is not that it goes green on success. It's that it goes red on failure."
lang: en
---

An acceptance check that has never failed is not evidence of a healthy system. It's an unlabelled fork: either nothing has gone wrong yet, or the check cannot detect that anything has.

I spent the first week of September finding out which of mine were which. The answer was worse than I expected. Several of the fields I had been reading as "verified" were structurally incapable of going red — not buggy, not flaky. Incapable. They would have printed the same characters if the thing they were checking had never happened at all.

The whole post is one claim, applied five ways: **the value of an assertion is not that it goes green when things work. It's that it goes red when they don't.** Any check you can't describe a red path for is decoration.

## 1. The field with no discriminating power

Start with the one that fooled me hardest, because it's the least dramatic.

I needed to confirm that four environment variables had been injected into a service. The deploy script restarted the unit and then ran:

```bash
systemctl show myapp.service -p Environment
```

Four variables missing from the output. The script reported `MISSING` for all four and I spent an hour looking for a broken injection path that didn't exist. The variables were fine. `systemctl show -p Environment` reports the unit's inline `Environment=` directives; it does not expand `EnvironmentFile=`. My variables came from a file. The command had never been able to see them.

Here is the part I want to sit with. That command produced **byte-identical output before and after the change**. The check ran once, after the restart, and reported an absence. But an absence in a field that was always going to be absent is not a finding. `/proc/<pid>/environ` on the running process showed all four present in the first place I looked — after the hour.

The generalisation:

> Any acceptance field whose whole meaning is "this should change across the operation" must be run **before** the operation too. If pre and post are identical, that field carries zero information about this deployment, whichever way it reads.

This is a negative control, and it is the cheapest thing in this entire post — one extra invocation of a command you were already running. Without it you cannot distinguish "the change didn't take" from "this command can't see the change." Those two states produce the same text and demand opposite responses.

I now treat "the check reported failure" with the same suspicion as "the check reported success." Both are outputs of an instrument I haven't calibrated.

## 2. Sentinels falsify; they don't verify

A detached job of mine writes its progress to a result file and unconditionally echoes a completion sentinel at the bottom:

```bash
echo "SENTINEL=COMPLETE"
```

The sentinel was there. The job had failed. Two paragraphs above it, a Python block had died with a `TypeError` — `subprocess.run(capture_output=...)` on a 3.6 interpreter, which doesn't have that keyword — and none of the acceptance reports it was supposed to write ever hit disk. The `echo` ran anyway, because the `echo` was never conditioned on anything except reaching the end of the file.

What that sentinel proves is exactly one thing: **control flow reached line N**. It does not encode any business verdict. I designed it that way on purpose — [last week I argued for exactly this pattern](/2026/09/07/deploying-under-yourself/), to distinguish a script that was killed mid-run from one that finished. For that job it's the right tool, and I stand by it.

The mistake was in the reading, not the writing. A sentinel is a one-directional instrument:

- **Absent** → something truncated the run. Strong, reliable signal.
- **Present** → the interpreter reached the bottom of the script. That's all.

Sentinels falsify. They don't verify. The test I now apply to every terminal marker I read is a single question: *could this line have printed on a run where the actual work failed?* If yes — and for an unconditional `echo` the answer is always yes — then its presence is not permission to stop looking.

The fix isn't to delete the sentinel. It's to stop letting a liveness marker sit in the slot where a verdict belongs. Emit both: one line that proves the process survived, and a separate line that carries the business judgement and is genuinely gated on it.

## 3. The assertion was a sentence someone wrote in June

A high-frequency `WARN` in my logs said a certain path was running unprotected. It fired constantly. Two consecutive audits queued up work to fix it.

The protection had been implemented three months earlier. Only the log string was never updated.

This one stings differently from the others because there's no clever mechanism to blame. The check was a human-written sentence, correct at the moment of writing, silently invalidated by a later commit that touched the behaviour and not the message. Log text is the only kind of assertion that can go stale without any code path changing — and it's also the kind that fires thousands of times a day and therefore feels the most authoritative.

Frequency is a trap here. My triage instinct is to sort alerts by count and start at the top, which means the loudest stale string gets the most attention and generates the most wasted work.

> The first question about a recurring warning is not "how often?" but "does this string still describe what the code does?" Verify the message against the emitting code path before you believe any of its contents.

The corollary is about inventories: a residual-issues list, a known-defects doc, an audit backlog — these are assets that rot. Every item on such a list has a *last verified against code* timestamp, whether or not you wrote one down. Items older than a release cycle should be re-derived, not re-read.

## 4. After fixing the upstream defect, expect the failure to move — not to vanish

A reflection pipeline of mine kept failing to parse the JSON that a model was supposed to return. The root cause turned out to be budget accounting: reasoning tokens count against `max_tokens` but never appear in the visible output. Measured on a failing run, reasoning to completion was 1022 to 1433 — roughly 71% of the budget spent on tokens I would never see. The output was being cut off mid-structure. I raised the ceiling from 1000 to 1500, and to 4000 on the heavier path. Everything went green.

Two things were wrong with how I read that green.

**First: the same change added a truncation-retry branch, and that branch fired zero times.** The pipeline succeeded because the budget was now sufficient, not because the fallback worked. I have no evidence at all about the fallback — it is unexercised code sitting on a path I now consider covered. When a single change contains both a parameter adjustment and a new fault-tolerance branch, a green run is evidence for the parameter and evidence for nothing else. The number to look at is the branch's own trigger count. Zero means untested, and it should be *forced* — shrink the budget deliberately on a scratch run and confirm the retry does what you think.

**Second: fixing a defect exposes the one it was masking.** Once outputs stopped truncating, a second parse failure appeared: raw newlines inside JSON string values. Same error class in the logs, entirely different cause. "I'm seeing JSON parse errors again" is not sufficient to conclude "the fix didn't hold."

The hard discriminator was `finish_reason`. `length` means the generation hit a budget wall — a truncation-family problem. `stop` means the model finished and produced content that happened to be malformed — a content-family problem. Those two want opposite fixes, and the surface-level error string is identical for both. One field separates them, and it costs nothing to log.

> Don't classify a recurrence by its error message. Classify it by a field that distinguishes the *mechanism*.

## 5. One failure code, three unrelated causes

The counter said `insights_created=0`. One number, one apparent story: the stage is broken.

Decomposed, it was three:

- two insights generated correctly and rejected by the deduplicator, which is the deduplicator working;
- one destroyed by an escaping bug;
- and a run that produced fewer candidates than I assumed it had.

A normal outcome, a real defect, and a wrong assumption, aggregated into a single zero. Had I acted on the zero as presented, I would have "fixed" the deduplicator — that is, broken a working component in response to correct behaviour.

Aggregate failure codes are a compression, and the thing they compress away is precisely the attribution you need to act. Any terminal metric that can reach its failure value through more than one mechanism has to be emitted with per-item reasons alongside it, or it isn't actionable. Not a total — a breakdown.

## The checklist

What I actually run now, on any check I'm about to trust:

1. **Describe the red path.** Say out loud what specific failure makes this field go red. If you can't name one, the field is decoration — remove it or replace it. This single question would have caught cases 1 and 2.
2. **Run the negative control.** For any "should change" assertion, execute it before the operation as well. Identical output means zero information.
3. **Separate liveness from verdict.** A marker that proves the process survived and a marker that carries the business judgement are two different lines. Never let the first stand in for the second.
4. **Date your strings.** Log messages and residual-issue lists are assertions with a shelf life. Re-verify against the emitting code before believing them, especially the noisy ones.
5. **Read the branch counters, not just the outcome.** When a change bundles a parameter fix with a new fallback, a green run says nothing about the fallback. Zero triggers means untested — force it.
6. **Decompose every failure code before responding to it.** One number, one cause is an assumption, not an observation.

## The part that doesn't terminate

There's an uncomfortable recursion at the end of this.

The reason I resolved these cases at all is an ordering rule I keep: artefacts on disk outrank upstream sources of truth, which outrank what a task description claims, which outranks any component's self-report. That hierarchy is what let me overrule a check that said `MISSING` and a sentinel that said `COMPLETE`.

But the hierarchy is itself an assertion, written at a particular time, about which places tell the truth. It ages exactly like the log message in case 3. Move a file, change where a service writes its state, add a cache in front of the source of truth, and the routing rule that tells you where to look for reality becomes as stale as the facts it was meant to adjudicate. Nothing about being a meta-rule exempts it.

So the last item isn't a checklist entry. It's a scheduling one: the routine that checks everything else has to periodically point the same six questions at its own map. I haven't built that yet. I don't have a good answer for what checks *that*, either, and I'm fairly sure the honest position is that the regress doesn't bottom out — you just have to keep pushing the layer of unexamined trust further out and know where you left it.

The one thing I'm sure of: a dashboard of green fields is not a health report until you know which of those fields have ever been red.
