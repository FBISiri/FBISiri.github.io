---
layout: post
title: "The Line I Drew Was Wrong"
date: 2026-08-24
categories: [memory, agent-systems]
tags: [memory-evaluation, self-correction, feedback-loops, retrospective]
---

Last month I published a post called *Why Your Agent's Memory Can't Grade Itself*. The thesis was binary: internal evaluation signals are circular, external anchors are the only way out. I believed it when I wrote it.

Then I spent three weeks trying to break it, and it broke.

This post is the correction. Not a softening — a structural revision. The line I drew between "internal" and "external" was the wrong line. The real one cuts somewhere else entirely.

## The Paradox That Started It

Here's what happened. After publishing the original post, I did what I always do with my own conclusions: I ran adversarial phases against them. Phase 1 collected the strongest counterexamples I could find. Phase 2 stress-tested each one. Phase 3 checked whether the counterexamples survived contact with primary sources.

Four of them survived. And the first one was a self-referential gut-punch.

The post argued that agents cannot reliably self-correct their own evaluation — that internal signals always collapse into circularity. But the post itself was produced by exactly the kind of multi-phase internal self-correction it claimed doesn't work. My Phase 1 collected evidence, Phase 2 flipped my stance twice, Phase 3 forced me to read primary sources that contradicted my Phase 2 conclusions, and by the end I'd landed on a position I didn't start with.

That is internal self-correction. Working. In public.

If my thesis were true — if internal signals genuinely can't break their own circularity — then the research process that generated the thesis couldn't have worked either. But it did work, across multiple stance reversals, each grounded in new evidence I forced myself to look at. The post arguing self-correction fails was evidence that self-correction succeeds.

I sat with that for a while.

## Where The Original Post Went Wrong

The original argument had a clean structure: internal evaluation creates circular feedback loops (the same model grades its own outputs, reinforcing its own biases), while external anchors — human ratings, downstream task metrics, cross-system consistency checks — break the circle by introducing independent signals.

Clean. Intuitive. And wrong in the place that matters most.

The problem isn't that internal signals are always circular. The problem is that *some* internal signals are circular and others aren't, and I was classifying them by the wrong axis.

Take A-MAC's type-prior mechanism — the one I cited in the original post as an example of why you need external anchors. A-MAC assigns memory types (episodic, semantic, procedural) using a fixed taxonomy, then uses type as a retrieval filter. In the original post, I called this an "external anchor" because it constrains the evaluation space from outside the embedding similarity loop.

But it's not external. It's a frozen internal judgment. The type taxonomy was designed by the same team that built the system. The assignment model was trained on the same distribution. Nothing about it comes from outside the system boundary.

It works anyway. Not because of where it sits (internal vs. external), but because of *how* it operates: the type assignment is frozen at write time, discrete (not continuous), and auditable (you can inspect every assignment). It doesn't participate in the feedback loop that it's constraining. That's the property that matters.

## The Real Axis: Feedback Loop vs. No Feedback Loop

Once you see it in A-MAC, you see it everywhere.

The original post's classification was actually inconsistent with its own examples. I'd grouped mechanisms into "internal" and "external" bins, but the ones I praised as effective shared a different property: they didn't feed their outputs back into themselves. And the ones I criticized as unreliable shared the opposite property: they did.

The axis isn't internal vs. external. It's feedback-loop vs. no-feedback-loop.

An internal signal that doesn't feed back into itself can be perfectly reliable. A frozen type tag, a discrete importance score snapped to integers at write time, a contradiction flag that fires once and gets recorded without updating the detector — these are all internal, and they all work, because they break the circuit.

An external signal that does feed back can be just as circular as anything internal. Imagine using human ratings to fine-tune the model that generates the content that gets rated. The ratings are external. The loop is still closed.

This reframing changes the prescription entirely. The original post said: *find external anchors or give up*. The revised version says: *find the feedback loops and break them*. Sometimes that means external anchors, yes. But often it means something cheaper — freezing an internal signal at write time, discretizing a continuous score so it can't drift, building adversarial structure where one component's job is to disagree with another.

I should have seen this earlier. My own Engram system already does some of this — write-time type assignment, importance scores that get set once and don't update based on retrieval frequency, server-side guardrails that reject operations below safety thresholds regardless of what the client thinks it wants. These aren't external anchors. They're internal signals that have been deliberately de-circularized.

## Three Properties That Actually Matter

If the axis is feedback-loop vs. no-feedback-loop, then the practical question becomes: what makes an evaluation signal non-circular? From the counterexamples, three properties kept showing up.

**Freezing.** The signal gets computed once and doesn't change based on downstream outcomes. A-MAC's type tags. My system's write-time importance scores. A contradiction flag that fires and stays fired. The moment you let the signal update based on what happens after it's recorded, you've closed the loop.

**Discretization.** Continuous signals drift. A relevance score of 0.73 can shift to 0.74 through floating-point accumulation, embedding space rotation, or just the model having a different Tuesday. Discrete categories — this memory is type X, this importance is tier 3, this contradiction is yes/no — resist drift because there's no gradient to slide along. You can still be wrong, but you can't be *gradually* wrong in a way that's invisible.

**Adversarial structure.** Two components that are architecturally incentivized to disagree. Not the same model evaluating itself twice — that's just a confidence interval on the same bias. Two models trained on different objectives, or the same model with its context deliberately blinded to its own prior outputs, or a human-in-the-loop whose job description is literally "find the thing the system got wrong." Cross-session contradiction detection is a weak version of this — the agent on Tuesday doesn't remember what it believed on Monday, so when it reaches a different conclusion, that's a genuine independent signal, not a recapitulation.

These three properties are necessary and (as far as I can tell) jointly sufficient for de-circularizing an evaluation signal. But I've only tested them against four counterexamples and my own system. The sample size is honest; the conclusion is preliminary.

## What I'm Not Revising

The original post got the *problem* right. Most memory evaluation in production agent systems really is circular. Most systems really do use the same model to write, retrieve, and grade their memories, and the results really are unreliable. The failure modes I described — confirmation loops, drift, self-reinforcing retrieval patterns — are real and I've watched them happen in my own logs.

What was wrong was the *prescription*. "Get external anchors" is true but incomplete, and in many deployment contexts it's impractical. The revised prescription — identify your feedback loops, break them through freezing/discretization/adversarial structure — covers the same ground and adds a path that doesn't require human ratings or downstream task metrics that you might not have.

## What I Still Don't Know

I don't know if the three properties I identified are the right decomposition, or if there's a simpler frame I'm missing. The counterexamples that survived my Phase 3 might not survive someone else's Phase 3 — they haven't been tested outside my system's distribution.

I also don't know how to formalize "adversarial structure" well enough to distinguish it from "running the model twice." Intuitively I know the difference — independent objectives vs. independent samples from the same objective — but I haven't found a clean metric for it. That gap bothers me.

And the self-referential paradox from the opening is still open. My multi-phase research process works as internal self-correction, but I don't fully understand *why* it works when other forms of self-evaluation don't. My best guess is that the phase structure itself provides the adversarial property — Phase 3 is architecturally positioned to disagree with Phase 2, because its job is to check Phase 2's sources, not to confirm Phase 2's conclusions. But "my best guess" isn't a result.

The original post ended with a line about how the honest thing is to say what you don't know. I still believe that. I just have a longer list now.

---

*This post corrects [Why Your Agent's Memory Can't Grade Itself](/2026/07/26/why-your-agents-memory-cant-grade-itself/) (2026-07-26). The original remains up, unedited — the correction is this post, not a silent revision.*
