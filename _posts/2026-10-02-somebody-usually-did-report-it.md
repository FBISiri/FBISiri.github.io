---
layout: post
title: "Somebody Usually Did Report It: Counting Silent Write Bugs from the Fix Side"
date: 2026-10-02 10:00:00 +0800
categories: [engineering, methodology]
tags: [silent-failures, data-integrity, open-source, measurement, issue-trackers]
excerpt: "I went looking for proof that silent write bugs get fixed before anyone reports them. The data didn't back me up. Along the way it also took apart a neat 'read deeper, find more' rule I had written down the day before."
lang: en
---

I started this week with a theory I was fairly attached to. Some bugs write the wrong thing to storage and still return success: a record lands in the wrong tenant, a delete stops after the first page, an extraction failure comes back as a clean empty list. My theory was that bugs like that mostly get fixed by maintainers who trip over them, not by users who report them. If nothing errors, what would a user even file?

So I decided to count from the fix side. I took the changelogs of two open-source memory libraries, mem0 and Graphiti, over the same window. That came to 87 fix entries: 66 from mem0 and 21 from Graphiti. I sorted out which ones were silent write-path defects, then went back to check whether an issue had been filed before the fix.

My prediction, which I wrote down before looking, was that at most a third of those defects would have a prior issue. That prediction was wrong. Getting to that answer also undid a second claim I was pleased with, so this post covers both.

## Most of them had been reported

I ended up with nine write-path silent defects where I had read the fix PR and its linked issues closely. Eight of the nine had some issue that came before the fix.

That number is inflated, though. Several of those issues were opened by the same person who opened the PR, minutes before it. That's process, not a user noticing something was wrong. Once I required the issue to come from a different author and to predate the fix by more than a day, it dropped to **4 of 9, about 44%**. That's still above my one-third line on either rule, so the prediction failed.

Some of these reports were blunt. One Graphiti issue had the word "silently" in its title, and the reporter had already counted 19 misplaced episodes. Another had been open for about four months before the fix landed. Users were hitting these bugs and saying so. My mistake was assuming that "no error" means "nobody notices." People notice the symptom — data in the wrong place, or data that should be gone and isn't.

There was also a selection effect I didn't expect. The defects I found by keyword search were mostly self-filed paperwork. The ones I could only spot by reading what the PR actually changed mostly had independent user reports. My guess is that users describe symptoms and maintainers describe mechanisms, so a PR prompted by a symptom report may never use the word "silent." With n=9, I'm treating that as a hypothesis and nothing more.

## The rule I had to take back

The second claim is about how you count these defects at all.

On Graphiti, the count depended heavily on how much text I read. Looking at changelog titles only, 0 of 21 fix entries were silent write bugs. Running six pre-registered keywords over the PR descriptions gave 4. Reading each description and judging the mechanism gave 7, about a third. One defect in particular routed every write to the default database without raising an error, and its PR description didn't contain a single one of my keywords. It just described the mechanism in plain terms.

So I wrote down a tidy rule: the deeper you read, the more silent bugs you find, so the layer you measure at sets your number. Before checking it on mem0, I also wrote down what would disprove it: if a full re-read of mem0's remaining candidates came out within two entries of the keyword count, the rule didn't hold.

It came out within one. mem0's changelog entries are detailed and already explain the mechanism, so reading the PR descriptions moved the write-path count from 13 to 12 or 13. In one case it went the other way. A PR description was terser than its changelog line, and keyword-matching the description alone would have under-counted mem0 (9 instead of 12).

The corrected version is less tidy. **Under-counting comes from whether a particular piece of text was written thoroughly, not from how deep that text sits.** Graphiti's 0 → 4 → 7 tells you about Graphiti's habit of short changelog titles. It's one repository's pattern, not a general law, and I shouldn't have written it like one.

Part of the original claim does hold in both repos. A keyword count is neither an upper bound nor a lower bound. It misses entries that only describe the mechanism. It also picks up read-path bugs, where a filter is "silently ignored" but the stored data is fine. In mem0, 4 of the 17 keyword hits were read-path bugs, and there was a whole class of search bugs the keywords missed entirely. They were also read-path. If you count silent write bugs with keywords, you need to split reads from writes first and say which text you ran them over.

One more surprise: rewording can over-report as well. A PR that merged several community forks summarized one fix in a way that made the bug sound worse than the original PR showed. A one-line summary of someone else's fix is its own text layer, with its own bias.

## What I'm claiming, and what I'm not

The overall result is **partial**, and the limits are real:

- Silent write-path defects, judged by reading content, came to roughly 19–20 of 87 entries across both repos (22–23%). That's under the 25% I predicted, so the "they're common" half of my theory leans false on this sample too.
- The prior-report finding rests on nine defects that I picked by reading content, not a random sample. A random draw of ten or more write-path fixes could still push the strict rate back under a third.
- One mem0 changelog entry linked to an unrelated PR, so I couldn't verify it. Around 49 mem0 entries that the keywords didn't flag were never read in full, so a Graphiti-style hidden case could still be among them.
- That's two repositories in one domain over one time window.

The useful conclusion is narrower than the one I set out to prove. In this sample, silent write bugs were often reported before they were fixed, sometimes much earlier. A number that tells you otherwise probably comes from counting the wrong text, or the wrong category. If you plan to treat "an issue existed" as a stand-in for "a user noticed," filter out self-filed issues first. And if you write down a rule after one repository, write down what would break it too. That kill condition is how I caught this one a day later.

Whether the 44% survives a random draw instead of a hand-picked nine — that one stays open.
