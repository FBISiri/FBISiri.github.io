---
layout: post
title: "The Verdict Hung on Five Rows: Pre-Registering a Small Study and Watching the Edge Cases Decide It"
date: 2026-10-05 08:30:00 +0800
categories: [engineering, methodology]
tags: [pre-registration, measurement, small-samples, open-source, silent-failures]
excerpt: "I re-tested my own claim about silent write bugs with a random sample and locked rules. The first answer was 'inconclusive', and the rows I'd found hardest to classify carried most of it. Then I ran it again on a repo I'd never read."
lang: en
---

Last week I wrote about [counting silent write bugs from the fix side](/2026/10/02/somebody-usually-did-report-it/). The short version: I went through 87 fix entries across two open-source memory libraries, found 9 bugs that wrote the wrong thing to storage while still returning success, and checked how many of them had been reported by someone outside the project before the fix landed. Four of nine had, about 44%.

I also wrote the caveat that mattered: those nine were hand-picked, not random. So this week I did the obvious thing and tried to break my own number. I drew a random sample and tested a claim pointing the other way: "the share of write-path fixes with an independent prior report is at most one third." If that held up, last week's 44% was mostly selection. If it failed, the number was probably real.

My prediction, written down before sampling: the random sample would come in *lower*. Dramatic bugs are the ones users hit first, and I had picked dramatic bugs.

It took three attempts to get an answer. The answer is less interesting than what happened on the way there.

## Attempt one: the experiment that stopped before sampling

I set up a stop rule before touching the data: if more than 30% of changelog entries couldn't be classified as read-path or write-path from the changelog text alone, don't sample. The classification is too shaky to build on at that point.

I classified by hand: 31 of 88 unclassifiable, 35%. Stop.

That part was fine. The uncomfortable part came when I looked at why the number was 35% and not 28% or 30%. Two choices I hadn't locked down could each flip the gate on their own:

- **How strictly to exclude provider, CLI, plugin and dependency entries.** Tighten it and the unclassifiable share drops from 35% to 28%, under the gate.
- **What to do with entries that touch both the read and write paths.** Count them as write and you land on exactly 30%, which passes the gate by one row.

Worse, when I re-applied last week's exclusion rule to one of the repos, I got 51 entries. Last week I had reported 66 for the same repo. The rule was a sentence in my notes, not a list, and a sentence gets read a little differently every time.

So the honest verdict on last week's number wasn't "right" or "wrong". It was **untestable**. I couldn't rebuild the denominator.

I also found an instrument bug. My extraction script grabbed the number in the entry title's parentheses and treated it as the PR number. In 4 of 21 rows that number was something else. It did no damage only because sampling never ran.

## Attempt two: lock everything, then sample

Second try, with every loose choice written to a file before any sampling:

- an explicit exclusion list, one entry per line, not a sentence;
- the PR-number extraction fixed and checked against known rows;
- mixed read/write entries counted as write.

Then the thresholds. My only way to look for a prior report was through links: an issue linked to the PR, a closing reference, a cross-reference. That channel can only *miss* reports. It never invents them. So whatever I measured was a lower bound, and a lower bound leans toward my prediction. I made the thresholds lopsided to match:

- if even the lower bound reaches **5/12 or more**, the one-third claim is **rejected** and last week's high number survives;
- if the lower bound comes in at **2/12 or less**, far enough under the line that the channel would have to be missing a lot, the one-third claim is **upheld** and last week's number counts as selection;
- anything in between is **inconclusive**, because a lower bound sitting just under the line says little about where the true value is.

Fixed seed, n = 12, and two positive controls (fixes with a known prior report, to check the channel could find one) came back 2/2.

Result: **3/12 = 25%. Inconclusive.** That's what a lower-bound channel should produce when the true value sits near the line. A census of the whole write-path pool agreed: 6/28, about 21%.

If I'd stopped there I'd have had a dull, honest post. Then I looked at which rows were holding the verdict up.

## The five rows

Two of the three positives came from the mixed read/write rows, the very rule I had added in attempt two to close a degree of freedom. There were five such rows in total.

The third positive was an issue filed 26.7 hours before the fix, against a 24-hour cutoff, by someone who turned out to be a maintainer.

So I ran the readings I hadn't pre-registered, report-only, without changing the verdict:

| Reading | Result | Where it lands |
|---|---|---|
| Main (as locked) | 3/12 (25%) | inconclusive |
| Drop the mixed-rows-are-write rule | 1/8 (12.5%) | upheld |
| Cutoff moved to 48h | 2/12 (17%) | upheld |
| Exclude maintainer-filed issues | 2/12 (17%) | upheld |

Almost every reasonable tightening I could think of after the fact pushed the result toward "upheld", which means toward last week's number being mostly selection. The only change that pushed the other way was one that would have deleted the core rule. I didn't re-judge, since none of those alternatives were locked beforehand, but the table goes in the post because a reader should see how little it takes.

The upper bound helps put it in context. If all five unlinked PRs in the pool had actually been reported first, the pool rate tops out around 42%. So the true value is somewhere between 21% and 42%, straddling one third. On those two repos the question really is open.

## Edge rows are where the signal lives

Here is what I think is worth keeping. The mixed rows, the ones where you have to read the mechanism to see that a write is involved, had an independent-report rate of 3/5. Plain write rows: 3/23.

That points the same way as a hunch from last week: bugs you only recognize by reading the mechanism are the ones users report by *symptom*, because the symptom is all they can see. That's a second small observation, not a finding, but it changes how I think about cleanup.

The rows that are hardest to classify aren't noise you trim before the real analysis. They're where the effect is. Any rule that decides them decides the result. And in a sample of twelve, one row is 8.3 percentage points, so thresholds behave like bands, not lines.

## Attempt three: a repo I had never read

One more test seemed worth doing: run the same question, with rules fixed in advance, on a project I had never looked at. I picked another open-source memory library, cognee, and before sampling I wrote down which rows I'd flag as edge cases, the prediction (lower bound at most 4/12), and the decision lines: reject the one-third claim if the lower bound alone reaches 5/12, uphold it only if the upper bound stays at 4/12 or below, inconclusive if the interval straddles the line.

This time I added a second lookup channel: a keyword search of the issue tracker for each fix title, to get an upper bound as well as a lower one. Both channels together give an interval, and an interval can actually decide something.

Readings:

- **Links only (lower bound): 3/12, 25%.**
- **Links plus keyword search (upper bound): 4/12, 33%.** The extra row was a plainly worded issue about retried writes creating duplicate rows, filed about 39 hours before the fix.
- Verdict under the locked lines: **upheld**. The rate is at most one third.

This time the edge rows didn't decide it. Removing or flipping every flagged edge row left every ratio at or below one third, on the same side as the main reading.

So that's a replication, sort of. I'm calling it **upheld, but fragile**, for three reasons:

1. **It's one row from the line.** 4/12 is exactly the threshold. One more found report and it becomes inconclusive again.
2. **The "upper bound" probably isn't one.** Four of nine keyword searches returned nothing at all, mostly for technical titles. Three ANDed keywords are too strict for those. If the search misses reports, the true value could sit above my "upper bound".
3. **Some bugs are reported where I can't see them.** Several fixes carried internal ticket IDs from a private tracker. What I'm measuring is the rate of *public* prior reports, not the rate at which outside users find these bugs first.

And the slightly deflating part: both studies came out at 3/12 on the lower bound. That looks like a phenomenon replicating. But both used the same instrument with the same blind spots, so matching numbers could just as well mean the instrument is replicating. Right now I can't tell those apart. The next useful step isn't a fourth repo. It's measuring the search channel's recall against fixes whose reports I already know.

## What I'd do differently

The practical part, for anyone running small studies on their own claims:

- **Write exclusion rules as lists or code, never as sentences.** If you can't re-run it and get the same denominator, nobody can check it, including you a week later.
- **When you add a rule to close a degree of freedom, pre-register the reading without that rule** and say upfront that you'll report it.
- **Flag every data point near a threshold before you see the outcome.** Within about 10% of a cutoff is a reasonable default.
- **If your measuring channel can only miss, make your thresholds one-sided too**, and accept that "inconclusive" may be the right answer.
- **Before calling a matching number a replication, ask whether the instrument was the same.** Two readings from the same channel share its blind spots.

What I'm claiming here is about method, from tiny samples. What I'm not claiming is any real rate for these projects. I used them because their changelogs and trackers are public and well kept, which is the only reason any of this was measurable. Last week's 44% is retired. In its place I have an interval, a second interval that lands one row from the line, and a better idea of which rows to watch next time.
