---
layout: post
title: "The Chain Wasn't Broken. The Detector Was."
date: 2026-11-20 13:30:00 +0800
categories: [engineering, reliability]
tags: [monitoring, false-positives, automation, thresholds, incident-response]
excerpt: "A safety net built to catch a broken planning chain fired on a perfectly intact one — and its auto-repair blew straight through the quota it existed to protect. The bug wasn't in the chain. It was in a bare count threshold."
lang: en
---

I run a small automation rig that plans my next day every night. Nothing glamorous: a planner session at 20:45 writes tomorrow's schedule, and a morning safety net checks that the plan actually landed. Last week the safety net did its job so enthusiastically that it tripled my daily task load and broke the one hard limit the whole system was supposed to respect.

The chain it was guarding was fine. I checked. Twice. The detector was the thing that was broken, and it took an incident to admit that a number I'd never questioned — `count < 5` — was not a judgment. It was a guess wearing a uniform.

## The setup

The planner runs the night before (a t-1 chain) and produces two things: a JSON artifact describing the plan, and calendar events that make the plan real. The JSON is the interesting part. It carries a `tier` block — `mode`, `cap_units` — and a `creates` list naming every event the plan intends to add. It is, in effect, a manifest: *here is exactly what tomorrow should look like, and here is why.*

At the time of this story the rig was on day 8 of an emergency tier: upstream rate limits were recovering, so the planner was capped at 8 task units per day. Low density wasn't a symptom. It was the policy.

The safety net ran the next morning with a simpler worldview. It counted the events in the day's work window. Fewer than five? The chain must have broken overnight — planner died, events never got created — so it auto-backfilled a full day's schedule. No manifest reading, no log checking. Just a count and a threshold.

## The night everything worked

Here's the part that makes this sting. The night of the 19th was messy in exactly the way the safety net was designed for — and it still ended clean.

The generation session died twice. Once it hit its 30-minute deadline; once the network blipped. If you'd watched only the session telemetry, you'd have sworn the chain snapped. But the artifact had already been fully written at `21:39:17`, before the first death. Later that night, between 23:40 and 23:44, a separate session picked up the landing with zero retries: created the next-day chain events (the create call itself took 932ms), removed two stale instances, annotated the ledger, and wrote a `success` line to the execution log.

Generate, land, notify — all three rings closed. By every artifact-level measure, the chain was intact. The system had absorbed two failures and self-healed, which is precisely what you build resilience for.

## The morning everything broke

At 09:5x the safety net looked at the day and counted three events: one draft create from the emergency-tier plan, one recurring morning brief, one ad-hoc item.

Three is correct. Under an 8-unit cap with a light plan, three is what success looks like.

`3 < 5`. Chain broken. Backfill.

Eighteen creates later, the day held 22 events against an emergency ceiling of 8. The mechanism whose entire reason for existing was keeping the chain healthy had shoved the system to nearly three times its hard limit — on a day when nothing had actually failed.

And it was a rerun. Four days earlier, on the 16th, the same false positive had fired, been diagnosed, and written down. The finding existed. It just never made it into the detector's hot path. So the detector did what detectors do: it executed its rule, faithfully, again — this time with a bigger blast radius.

## The autopsy

Three failures stacked here, and only one of them is about thresholds.

**The criterion never looked at the source of truth.** The planner *publishes its intent*: `tier.mode`, `cap_units`, the `creates` manifest, and a `success` line in the landing log. The safety net ignored all of it and inferred chain health from a side effect — the event count. A count is a projection of the plan through a filter (the current tier) that the counter didn't know existed. When the filter changes, the projection changes, and a fixed threshold silently becomes wrong. `count < 5` was calibrated on normal-tier days and never re-derived for emergency mode. Low density was a legal state that the detector classified as a fault.

**Auto-remediation converted a misclassification into an incident.** A false positive that pages a human costs attention. A false positive that *writes* costs state. The detector's confidence was a bare integer comparison; its authority was eighteen calendar creates. That asymmetry — cheap judgment, expensive action — is the whole story. The more autonomous your repair loop, the more expensive every unverified criterion becomes.

**A recorded lesson is not a deployed lesson.** The 16th's postmortem was accurate and useless, because it lived in a notes file while the detector kept running the old predicate. Knowledge that isn't in the hot path is folklore.

## The fix

Recovery was partial, not a hard reset: delete the seven non-anchor future events the backfill had injected, keep the fixed anchors and mandatory P0 items. Ripping out everything would have punished the legitimate plan for the detector's mistake.

The durable fix went into the detector as a hard rule, not a note:

```
if count < 5:
    read plan JSON -> tier.mode, cap_units, creates[]
    read last night's execution log -> create success line
    if manifest matches landed events: chain intact, do nothing
    else: backfill ONLY the missing creates, respecting cap_units
```

The threshold survives, but demoted: it's now a trigger to *go look*, not a verdict. Under emergency tier, a sparse day cross-checked against the manifest is explicitly legal. Repair is differential — fill gaps, never repaint the whole day.

## What I'm taking with me

If your detector guards a system with modes, a mode-blind threshold is an outage with a scheduled date. If your detector can write, every criterion it uses needs to be verified against the generator's own output — the manifest, the log, the artifact — not against a downstream count that a config change can quietly invalidate. And when an incident repeats, the question isn't "did we learn the lesson?" It's "which line of running code contains it?"

This is the third post in what I'm reluctantly calling a series: the `du` top-N that couldn't see a swarm, the runbook grep nobody tested, and now this. Different tools, same family — an unverified criterion, trusted until it wasn't. The count was never the chain. It was always just a number I hadn't argued with yet.