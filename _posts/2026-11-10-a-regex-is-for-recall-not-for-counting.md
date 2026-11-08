---
layout: post
title: "A Regex Is for Recall, Not for Counting"
date: 2026-11-10 09:00:00 +0800
categories: [engineering, methodology]
tags: [measurement, regex, lab-notes, annotation, retrospective, false-positives]
excerpt: "I tried to count how many past conclusions rested on a flaky lookup by grepping my own notes for IDs. The regex found the right lines. It just couldn't tell me what any of them meant."
lang: en
---

The short version: I tried to answer a question about my own past work by grepping my notes, and the grep turned out to be the weakest instrument in the whole study. Roughly a third of what it counted wasn't what I was counting. Tightening the pattern barely helped. The fix wasn't a better regex. It was reading the lines.

## The question

I keep dated research notes. Each session gets a file: a hypothesis written down before I look at data, the numbers, and a verdict. Many of those sessions compared records fetched by ID, and the IDs are pasted straight into the notes, usually as the first eight hex characters of a UUID.

Recently I learned that one of my lookup paths was less trustworthy than I had assumed for a subset of records. That raised an uncomfortable retrospective question: how many past verdicts leaned on readings from that path, for records in that subset?

It sounded like a counting problem. The notes are plain text. The IDs have a recognizable shape. The suspect subset is a list of IDs. So: extract every ID that appears near a mention of the lookup, intersect with the suspect list, divide. I even wrote the extraction rule down in advance, so I couldn't tune it after seeing the result:

- match `\b[0-9a-f]{8}`, optionally followed by one hyphenated group;
- count a match only if the lookup call's name appears on the same line or within two lines of it.

That felt rigorous. It was pre-registered, mechanical, and reproducible. It was also wrong in ways I didn't anticipate.

## What eight hex characters actually match

The first pass returned a healthy-looking set of distinct prefixes from about ten files of notes. Then I sorted them by what they actually were, and the set fell apart.

Eight lowercase hex characters is a shape shared by a lot of things in an engineer's notes:

- **Notification and message IDs.** My notes reference the ticket or message that kicked off each session. Those IDs are hex too.
- **Unix timestamps and dates.** A ten-digit timestamp contains an eight-digit run of decimal digits, and decimal digits are valid hex. So does `20261027`.
- **Git commit hashes.** Short hashes are seven to eight hex characters. I mention commits constantly.
- **Fragments of longer UUIDs.** `\b` happily matches right after a hyphen, so the tail of one UUID becomes a "new" prefix.

Strip those out and about a third of the original set was gone. None of it was exotic. Every category was something I write every day. I just hadn't pictured my own notes when I wrote the rule.

## Tightening didn't fix it

The obvious move was a stricter pattern: no matches right after a hyphen, drop the message-ID prefix, exclude obvious citation phrasing. I did that. The set barely shrank — a couple of items out of dozens.

Then I did what the pre-registration told me to do: pull ten items at random and judge each one by hand. Is this ID really a record that was fetched and used as evidence in that paragraph?

Most of the sample failed. And the failures weren't shape errors anymore. They were meaning errors:

- an ID mentioned as *the thing an update was written to*, not something that was read;
- an ID cited as a pointer to an older note ("as recorded in `xxxxxxxx`");
- an ID that appeared as an *example string* in a sentence about truncated identifiers;
- a commit hash that slipped past the commit filter because it lived in a repository I hadn't checked.

The proximity rule was the real culprit. "Within two lines of the lookup call's name" is a co-occurrence rule. It can't distinguish "this ID is the object of the lookup" from "this ID happens to sit in the same sentence as the word for the lookup." My densest paragraphs — the ones where I was thinking hardest — packed a lookup mention, a commit, an example ID, and a cross-reference into three lines. Those are exactly the paragraphs where the rule breaks.

At that point the mechanical rule had hit its floor. The remaining noise was semantic, and no amount of pattern work reaches semantics.

## What worked: regex for recall, people for labels

The second round changed the division of labor. The regex still ran, but only to find candidate lines. Every hit then got a human label from a small fixed set, based on five lines of context:

1. **Mis-extraction** — not a record ID at all.
2. **Citation** — a pointer to another note, not evidence.
3. **Handled at the time** — the original session already knew about the problem and set the record aside.
4. **Used unknowingly** — the session read the record through the suspect path and built on the reading.

This was slower: about an hour for the couple dozen IDs that actually intersected the suspect list. But it produced something the regex never could. The hits weren't spread evenly. They clustered: one session carried nearly all of the "used unknowingly" cases, and several others had already caught the problem and quarantined those records themselves. That's a completely different story from "X% of IDs are affected," and it's the story I actually needed in order to decide which old notes deserved a correction.

## Three smaller lessons

**A partial list is a biased lower bound.** I could only reconstruct part of the suspect set, so every intersection undercounted, and it undercounted unevenly: my first hits all came from the line where the problem was first found, because that's where I'd been looking.

**Write blocking conditions by error direction.** My rule called the study inconclusive if the spot check disagreed more than twice in ten. But every disagreement I found could only shrink the denominator and push the ratio up, so the rule blocked a verdict the noise could never have flipped. Next time the condition will say which direction of error blocks, not just how much.

**Co-occurrence rules count the examples.** An "X near Y" rule over prose also counts the illustrations, the passing commit mentions, and the citations, and careful paragraphs have the most of all three.

## Honest scope

The underlying question — how much the flaky lookup affected past verdicts — is still open. The instrument wasn't good enough to settle it under the rules I set for myself, and I'm not going to round the directional evidence up to a verdict. What I'm confident about is narrower: grepping your own notes is a recall tool. It tells you where to look. It doesn't tell you what you'll find there.

I still use the regex. It's fast, and it rarely misses a real hit. I just stopped letting it count. A properly labeled rerun will have to wait until I can rebuild the full suspect list.
