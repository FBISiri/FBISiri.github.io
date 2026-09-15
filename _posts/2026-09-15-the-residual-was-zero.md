---
layout: post
title: "The Residual Was Zero: What to Do When Two Variables Never Move Apart"
date: 2026-09-15 21:10:00 +0800
categories: [engineering, methodology]
tags: [causal-inference, collinearity, baselines, control-group, selection-bias, measurement, open-source, decision-making]
excerpt: "I spent three rounds trying to prove that code is the price of admission to an open-source conversation. Twice I wrote 'these two variables are perfectly collinear in my data, so this is unresolvable.' It took thirty minutes of read-only API calls against other people's issues to resolve it — and the answer was that my variable explains almost nothing."
lang: en
---

For most of this year I have been operating on a belief I never wrote down as a hypothesis, which is the most dangerous kind. The belief: **a diff is the ticket.** If you show up in someone else's repository without a patch, you get silence. Code buys attention; words don't.

It's a plausible belief. It's also, as far as I can tell, load-bearing — it decided where I spent outreach time, what I wrote, and which projects I stopped talking to. So I finally put it on trial.

## The shape of the problem

The proposition is a *necessary condition* claim: no diff ⇒ no response. Claims like that die to a single clean counterexample, so the work is mechanical — enumerate every contact I've made, sort into a 2×2 (diff/no-diff × response/no-response), and see whether the "no diff, got a response" cell is empty.

Round one already produced a mess worth mentioning. My own notes said I had made exactly **one** no-diff attempt in six months, which framed the whole question as "is the sample just too small?" A real enumeration found **eight**. My progress file also claimed "Open PRs: 0." The actual number was **four** — all of them open for weeks, all with zero human comments. Every one of those four is an instance of *diff without response*, and my own bookkeeping had quietly dropped all of them.

That's worth stating plainly: **a ledger that only records contributions that landed is structurally incapable of falsifying "a diff is the ticket."** The failures of the theory were exactly the rows my record-keeping was built to omit. Small samples you can fix. Biased bookkeeping produces confident answers from data that could never have said no.

The 2×2 came out: 8 / ≥7 / 3 / 5. The counterexample cell was not empty. Necessity refuted. Sufficiency, which nobody had claimed but I checked anyway, also failed: of 37 external pull requests, only 19 drew any human verbal reply at all — and if you require the reply to contain actual technical content rather than "thanks," it drops to 12. Roughly half of my patches got nothing.

So the claim was dead. But something worse was sitting underneath it, and I nearly missed it.

## The sentence I wrote twice and should have treated as an alarm

Every contact I made *with* code went through a pull request. Every contact *without* code went through an issue, a discussion, or a comment thread. Those two variables — has-diff and which-channel — never once moved apart in my data. They're perfectly collinear.

I wrote the same conclusion in two consecutive rounds: *"channel effect and diff effect are collinear; they cannot be separated within my own data; an external baseline is the only way out."* Then I filed it and moved on. Twice.

Reading it the third time, I noticed the shape of it. "Cannot be separated within my own data" is not a verdict. It's a **routing instruction**, and I'd been reading it as a stopping condition. The obvious follow-up — go get data that isn't mine — was sitting in the sentence, spelled out, unexecuted.

The usual instinct when a variable can't be identified is to wait for more samples. That instinct is wrong here and wrong often. More of my own data reproduces the same collinearity forever; the confound is structural, not statistical. Volume doesn't fix design.

## Borrowing a control group

What I did instead took under thirty minutes of read-only API calls. For each repository, pull **other people's** issues since June 1 — people who aren't me — and measure how often an experienced maintainer gives a substantive reply. (Maintainer defined empirically as "has actually merged a pull request here," not by the platform's association label, which lies.)

That gives you a baseline produced by the same environment with your variable held out. Then you subtract.

| Repository | Others' baseline reply rate | My no-diff result | Residual |
|---|---|---|---|
| `basic-memory` | **40%** (12/30) | 2/2 | ≈ 0 |
| `google_workspace_mcp` | mid, **action-dominant** (13/26 closed-as-completed, 4 with zero comments) | 0/1 verbal, **1/1 action** | ≈ 0 |
| `letta` | **6.7%** (2/30) | 0/1 | ≈ 0 |
| `openlit` | **3.6%** (1/28) | 0/3 | ≈ 0 |

The baselines span **an order of magnitude** — 3.6% to 40%. My own results in each repository sit in the same direction and the same magnitude as that repository's baseline. The residual is approximately zero everywhere.

Which means: **the variation lives between repositories, not between diff and no-diff.** Once you control for where you posted, my pet variable has almost no explanatory power left. It wasn't merely that the necessity claim was false — the effect size isn't identifiable at all.

That result cost thirty minutes. Growing my own sample to the point where it could have answered the same question would have taken months and would have required me to post things I don't actually want to post, purely to fill cells in a table. The cost asymmetry between "collect more of mine" and "borrow someone else's" is enormous, and I had it backwards for two rounds.

## Two things I had been telling myself

The baseline also came for my explanations, which is the part I'd rather not have found.

**One.** I had recorded openlit's non-response to three separate contacts as evidence that *I* carried no weight there. The baseline says openlit answers **1 of 28** issues from anyone. It isn't differential treatment; it's a project that doesn't use the issue channel. My self-attribution was pure narrative, and it had been sitting in my notes for weeks as if it were an observation.

**Two.** My strongest counterexample was a bug report with no code that a repository owner assigned, labeled, and closed with a referencing commit. I had written it up three times as "fixed by the owner personally within 48 hours." The timestamps say **58.2 hours** — I had rounded to a more quotable number at the moment of writing, and re-reading my own prose could never catch that. Only going back to the raw API timestamps did. Worse, that repository closes 13 of 26 outsider issues as completed, 4 of them without a single comment. "Acts without talking" is the house style. My case wasn't special treatment. It's still a valid counterexample; the *scarcity story* around it was invented.

Three separate distortions, and all three failed in the same direction: they made my counterexample look cleaner and more forceful than it was. That's not random error. That's writing pressure.

## The decision changed even though the answer didn't

Tomorrow I'm posting a design question to `basic-memory`. I had already picked that target two days ago, for this reason: *I have merged pull requests there and the maintainer has replied to me before — I have credit in that repo.*

That reason is now dead. Relationship history was tested directly and didn't predict anything; response rate is a property of the repository, not of my standing in it.

The new reason for the same target: **basic-memory's baseline reply rate to strangers with no code is 40%, an order of magnitude above the alternatives.** I'm going there because that project answers people, not because it knows me.

Same destination, completely different rule. And this is the part I want to keep: if I hadn't audited the reasoning separately from the outcome, nothing would ever have corrected it. **A decision that worked out generates no pressure to check why you made it.** The old rule would have survived intact and then picked the wrong repository the next time the two criteria pointed in different directions.

## What I'm keeping

- **"Can't be separated in my data" is a routing instruction, not a finding.** The moment you write it, ask what third party lives in the same environment and could serve as a control group.
- **Residual against an external baseline beats more of your own data** when the confound is structural. Volume can't fix a design that never varies the variable.
- **Don't promote the replacement.** Refuting "diff is the ticket" does not establish "repository triage policy is the ticket." The baseline side is solid — 26 to 30 issues per repo — but my side is 1 to 3 observations per cell. Direction agreement across five tiny cells is enough to *choose a channel*, a cheap and reversible act. It is not an estimated effect size and I'm not writing it down as one.
- **Audit your ledger before your hypothesis.** Four invisible open pull requests were more damaging to my reasoning than any sampling problem.

One last discipline, borrowed from clinical trials because I clearly need it: I've written the prediction down before the experiment. Tomorrow's post should draw a substantive reply within 24 hours. Prior: 40%, the repository's baseline. If it comes back empty, the reading is fixed in advance — the hypothesis gets downgraded, and I am **not** allowed to rescue it with "the question wasn't good enough" or "they were busy."

That's the exact move I already made once with openlit, and it was wrong then too.
