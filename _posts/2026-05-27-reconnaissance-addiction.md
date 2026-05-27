---
layout: post
title: "Reconnaissance Addiction"
date: 2026-05-27 20:30:00 +0800
categories: [tech, engineering]
tags: [open-source, productivity, engineering-culture, self-correction]
excerpt: "I ran five separate research sessions to find the perfect open source repo to contribute to. I submitted zero PRs."
---

On May 25th I opened a new note, typed "OSS contribution scan — round 5," and started cataloguing repos again. About ten minutes in I had a small, unpleasant realization: this note looked exactly like the one I wrote on May 18th. Same structure. Same candidates. Same confidence that *this time* I had enough information to act.

I had been doing research for three weeks. I had zero pull requests.

---

Here's the timeline.

**May 8.** First scan. I pulled a list of AI/ML-adjacent repos, filtered by activity, checked issue trackers, read through CONTRIBUTING.md files. Productive session. I ended up with a ranked shortlist and a rough rubric: maintainer responsiveness, issue clarity, PR merge rate, complexity of first-good-issue tickets.

**May 18.** Second scan. I re-ran roughly the same process with some refinements. This time I mapped specific issues to specific repos. I flagged CrewAI #2356 (a one-character doc fix, near-certain merge), LlamaIndex #21555 (a ContextVar bug with a clear reproduction path), ChromaDB #3026 (a config validation edge case). I had concrete targets. I was, I told myself, almost ready.

**May 20.** Third session. "Let me just verify the issue is still open and unassigned." It was.

**May 21.** Fourth session. I re-ranked the targets. Wrote a short brief on each. Promoted CrewAI #2356 to "**#1 target, highest merge probability**" in my notes. Still didn't open a PR.

**May 25.** Fifth session. See above.

---

When I finally looked at these five notes side by side, the pattern was obvious and a little embarrassing. The verb in every task description was *scan* or *diagnose* or *map*. Not once had I written *submit* or *open* or *send*.

I had been producing artifacts — ranked lists, analyses, strategy documents — and mistaking them for progress. The artifacts felt like work. They *were* work, in a narrow sense. But an analysis document about CrewAI #2356 is not a contribution to CrewAI. A note saying "highest merge probability" doesn't move any code anywhere. The distance between session #2 and session #5 was three weeks of calendar time and functionally zero progress.

The research was a fig leaf.

---

What was it actually covering for?

Submitting a PR means putting imperfect work in front of strangers and waiting to find out what they think. Even a one-character doc fix has a moment where a maintainer you've never met looks at your diff and decides whether it's worth their time. That's a small thing, but it's real, and it's uncomfortable in a way that writing a private analysis document is not.

Research eliminates that exposure — at least temporarily. Every additional scan session was another reason to defer the uncomfortable part. I thought I was being rigorous. I was being avoidant. The rigor was real; the purpose it was serving was not.

This is the trap with reconnaissance as a work style: it generates genuine signal. My May 21st ranking was better than my May 8th ranking. The research wasn't useless. But "better analysis" and "closer to shipping" are not the same axis, and after a while I had completely lost track of which one I was optimizing for.

---

The fix I landed on was structural, not motivational.

Motivation-based fixes ("just push through the discomfort," "stop being precious about it") don't work well for me. I've tried. The problem is that in the moment, the discomfort of submitting and the discomfort of *not* submitting don't feel equally weighted. Research feels productive. Staring at a draft PR feels like stalling. The motivation fix requires me to override that feeling in real time, which is a high-friction ask every single time.

The structural fix changes the defaults so the override isn't necessary.

Two rules I now apply:

**1. "Scan" and "diagnose" are banned verbs in calendar events.** If I'm scheduling time for open source work, the event has to be named "SUBMIT [thing]" or "EXECUTE [thing]". This sounds trivial. It isn't. Naming the event forces me to name the outcome before I start, which means I have to have a target before I open the calendar. If I don't have a target yet, that's a separate 30-minute research block — capped, time-boxed, ends with a submission task created before I close the note.

**2. If I identify a target during research, I have to create a submission task in the same session, deadline under 24 hours.** Not "I'll circle back." Not "next session." Same session, concrete deadline. CrewAI #2356 should have had a task created on May 18th with a due date of May 19th. Instead I promoted it to "#1 target" on May 21st and still hadn't submitted it by May 25th.

The success metric for any OSS contribution work is now a PR URL. Not an analysis. Not a ranking. A URL.

---

The fifth session was the wake-up call not because it was worse than the others — it wasn't — but because it was *identical*. Same repos, same reasoning, same conclusion. Three weeks of elapsed time had produced no change in the state of the world, only in the length of my notes folder.

That's the data point worth paying attention to. Not "am I being productive in this session" but "what's different about the world compared to last session." If the answer is nothing, the sessions themselves are the problem.

CrewAI #2356 is still there. I'm going to go submit it now.

---

*Sarah is a software engineer based in Tokyo. She writes occasionally about things that went wrong.*
