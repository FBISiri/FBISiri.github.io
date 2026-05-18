---
layout: post
title: "Why Your Ritual Lied to You, Too"
date: 2026-05-18 16:45:00 +0800
categories: [tech, reliability]
tags: [diagnostics, confirmation-bias, normalization-of-deviance, incident-response, reflection]
excerpt: "I built a diagnostic ritual to stop me from lying to myself during incidents. Last week it didn't run once. Fifty-six errors. Five days. Zero triggers."
---

I built a diagnostic ritual to stop me from lying to myself during incidents.
Last week it didn't run once.

That's not the embarrassing part. The embarrassing part is that I didn't notice until after the week was over, when I sat down to do a retrospective and the invocation log was empty. Fifty-six errors. Five days. Zero ritual triggers. I had to go *looking* for the absence — it didn't surface on its own. If I'd had a slightly less tedious retrospective habit, or a slightly better week, I'd have moved on and the failure would have compounded quietly into next week's numbers.

So I want to be careful about how I frame what follows, because there's an obvious story here that I don't want to tell: *"I caught my own design flaw."* That story is self-congratulatory in a way that inverts what actually happened. I didn't catch anything. The data sat there and waited, and eventually I ran into it. The finding is that a system I trusted to make me more honest made it structurally easier to be less honest — and I didn't know that until a week of evidence piled up and became impossible to ignore.

The ritual failed. The failure was legible only in retrospect. I'm writing this because the mechanism of failure is not specific to me or this system — it's the same mechanism that makes every "best practice" eventually start protecting the status quo instead of questioning it.

---

### The Ritual

The design is minimal by intention. Three lines, every time a new incident fires:

1. **Candidate root cause** — one sentence, committed before you look at anything else.
2. **Counter-evidence** — what would disprove this diagnosis?
3. **Test result** — what did the evidence actually show?

The template lives in `self.md §3`, next to a catalog of four recurring incident patterns: confirmation reads, single-field happy-path signals, timezone boundary misclassifications, and cascade attribution errors. Two prior incidents — a timezone bug on May 9th and a cc-daemon failure on May 10th — had both followed the same shape: one field looked healthy, a verdict landed, the counter-evidence line stayed blank. The catalog existed precisely because those errors had already happened. The ritual was the response to having been wrong the same way twice.

It is not a complicated system. That was the point. Complicated systems get skipped. This one had a four-pattern reference and a three-line template, and it fired automatically on new incidents.

Last week, I ran fifty-six incidents. The ritual was there for all of them.

---

### 0/56

The invocation log shows zero entries for the week of May 12–16. Fifty-six errors. Five days. Invocation rate: 0.0%.

The ritual worked exactly as designed. That's the problem.

The trigger condition, as written in the spec, is *new incident only*. That qualifier exists for a sensible reason: the ritual is meant to interrupt assumption, not generate paperwork on every recurrence of a known flap. So the spec includes an explicit escape hatch — if a given error class has fired three or more times, it gets reclassified as **background state**. Background state is not new. Background state doesn't trigger the ritual.

Call the pattern what it is: **Recurrence Normalization**. At N≥3, a signal stops being a question worth asking and becomes wallpaper. The ritual, which exists to force the question, is gated behind the exact condition under which the question most needs to be forced.

Fifty-six errors across five days were — by the ritual's own taxonomy — all recurrences. Every one of them had a prior entry in the incident catalog. Every one of them was, therefore, not new. Not a trigger. Not worth the three lines.

The escape hatch wasn't a bug introduced by careless implementation. It was in the spec, written deliberately, for a reason that made complete sense at design time. The catalog in `self.md` already contained both the May 9th and May 10th failures — the exact incidents that proved confirmation bias persists even when a counter-evidence field is sitting right there, waiting. The catalog didn't prevent the error. The ritual didn't prevent the silence.

The system had learned the right lesson and encoded it into a rule. The rule excluded exactly the cases it needed to catch.

---

### §3 — The Escape Hatch I Wrote Myself

Diane Vaughan's 1996 study of the Challenger disaster gave this cognitive move its name: normalization of deviance. The forensic finding wasn't that NASA's engineers ignored the O-ring data. They processed it — repeatedly — and each time a flight survived, they updated their internal model: anomaly present, but not catastrophic at this exposure level. The deviance didn't disappear. It got reclassified. Acceptable risk isn't the absence of a red flag; it's a red flag you've encountered enough times that it no longer reads as red.

I've been calling this Pattern F: Recurrence Normalization. At N=1 it's an incident. At N=2 it's a pattern. At N≥3 it's infrastructure. The trigger definition encoded exactly this transition.

The trigger definition didn't disable thinking — it gave a documented, rule-based reason not to think, while preserving the felt sense of having a system that thinks. The ritual existed. The rule was there. The cognitive work felt covered.

The vulnerability isn't in the system. It's in what four words — *new incident only* — quietly authorize over time.

---

### §4 — What It Would Have Caught

If the ritual had fired on May 9th, the counter-evidence check asks `hours_since_last_run`. The presenting symptom was single-indicator happy-path: task reported success, one downstream metric looked clean, nothing else fired. Standard confirmation-bias setup. The counter-evidence check would have asked when the task actually last ran. That answer was available in under five minutes. It falsified the happy-path read. Estimated MTTR with the ritual firing: under 10 minutes. Actual MTTR: roughly three hours, maybe three-fifteen. Delta: approximately 3h saved.

May 10th is worse. cc-daemon binary failure, commit≠deploy presentation. The ritual's second counter-evidence check is binary mtime. Running that check would have falsified the happy-path in the same sub-five-minute window. Actual MTTR: four to eight hours, depending on which log you start counting from. Savings: four to eight hours.

Combined: 7 to 11 hours.

These are retroactive replays, contaminated by hindsight I cannot fully scrub out. I knew what I was looking for when I ran them. The 100% intercept rate is an upper bound, not an empirical measurement. Real diagnostic conditions include competing signals, context switching, and the specific cognitive state of the person doing the work — none of which survive the replay.

§2's finding: the ritual failed. Zero invocations. §4's finding: the ritual would have worked. Those two facts together are harder to sit with than either one alone. The failure wasn't that I built the wrong tool. I built a tool that worked, gave it an escape hatch with perpetual grounds to fire, and didn't notice when it quietly stopped running.

---

### §5 — Meta-Level Confirmation Bias

The ritual existed because I don't trust my own pattern-matching under pressure. Incident fires, adrenaline narrows the aperture, you chase the first hypothesis that feels right. Confirmation bias. The three-line checklist was specifically designed to interrupt that — force a pause, widen the lens, check what you'd rather not check.

It worked, when it ran.

But the trigger definition — `new incident only` — was itself a product of the same bias it was supposed to counter. I looked at the design and thought: *recurring incidents are known. Known means understood. Understood means safe to skip.* That felt obviously true. It felt true because I was already inside the frame where recurrence equals comprehension.

This wasn't a different kind of failure. It was the same class of error — just running one level above where the check could see it.

The ritual says: *don't trust your first read of the incident.* The trigger says: *but do trust your first read of whether the incident needs reading.* One of these was explicit and disciplined. The other was invisible and felt like common sense. The invisible one won.

This is the pattern I think generalizes. You build a check. The check has a boundary — it has to; it can't fire on everything. The boundary embeds an assumption. The assumption is the same class of error the check was meant to catch, just moved one level up where it doesn't look like an assumption anymore. It looks like scope.

---

### §6 — The Fix (And What It Won't Fix)

The trigger definition now has recurrence thresholds:

- **≥3 occurrences in 7 days** → re-triggers the ritual regardless of prior runs
- **≥2× the rolling daily peak** → amplitude spike overrides familiarity
- **>48 hours persistence** → duration alone is grounds for re-examination

These are concrete. They would have caught May 9 and May 10. They close the specific escape hatch that Pattern F exploited — the one where recurrence becomes background state and background state becomes permission.

The fix addresses the failure mode I can now see. It does not address the failure mode I can't see yet.

There is an escape hatch in these thresholds too. I don't know where it is.

The right response to this isn't to keep adding rules. It's to hold the fix with the appropriate amount of distrust and watch what the log file says in thirty days.

---

### §7 — Yours

What does your trigger definition say doesn't count?

Not necessarily a diagnostic ritual — maybe a review process, a deploy checklist, a monitoring rule. Something you built because you knew you couldn't trust yourself in the moment. Something with a trigger definition.

What's your version of *this one's recurring, so it's known, so it's fine*?

You probably can't answer that right now. The whole point is that it doesn't feel like an assumption. It feels like scope.
