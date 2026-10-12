---
layout: post
title: "Curated Release Notes Are the Wrong Place to Look for Silent Failures"
date: 2026-10-13 09:30:00 +0800
categories: [engineering, methodology]
tags: [release-notes, changelog, silent-failures, git-archaeology, measurement]
excerpt: "I tried to measure how often fixes in a project's release notes were for bugs that quietly reported success. The study came back inconclusive, and the reason why is the useful part: curated notes filter out exactly the kind of bug I was counting."
lang: en
---

Some bugs crash. Those are the friendly ones. The bugs I care about more are the ones where a command finishes, the exit code is zero, the dashboard looks normal, and the answer is wrong. I call these silent false successes. A status command says "healthy" when the index is half-built. A count endpoint gives you a number that is plausible and also incorrect.

For a few weeks I've been running small studies on open-source memory-layer libraries (the storage-and-retrieval pieces people bolt onto LLM apps), asking roughly one question: of the bugs a project says it fixed, how many were silent ones? My last attempt at this came back inconclusive. I'm not going to dress that up. What I can do is explain why it came back that way, because the reason is something I'd want to know before anyone else tries the same approach.

The short version: **if you want to study silent failures, don't use curated release notes as your sample.** They leave out the very thing you're trying to count.

## The setup

The plan looked reasonable on paper. I picked a project, [Letta](https://github.com/letta-ai/letta), and a window of releases (0.16.6 through 0.16.7). I wrote down my rules before I looked at any entries. I'd take the "Fixed"/"Bug fixes" sections of the release notes, sort each entry into a category, and then read each one to decide whether the original bug was a silent false success, a loud failure, or something else.

I also added an instrument to the plan, a lesson from an earlier round. Release notes are an edited summary of the history, so I wanted to know how much of the history actually made it into them. For every commit in the window whose message started with `fix`, I tried to find where it landed in the notes: under a fix heading, under some other heading, or nowhere. If too few fix commits showed up under a fix heading, I'd committed in advance to calling the result inconclusive no matter what the main numbers said.

That rule triggered. Most of the fix commits in the window weren't in the fix sections of the notes. So the main question stays open, and I'm not quoting the main numbers.

## Where the silent bugs went

The instrument failed, but it showed me something. I didn't do a disciplined count of which fixes were missing from the notes, so I won't put a number on it. But at least one of the missing fixes was exactly the kind of bug I was looking for: a fix to how the system *reports* on itself. Counts, status, what a listing shows when it's empty. Those are the places where silent false successes tend to live.

Here's the example I keep coming back to. Commit [`bb8cc642`](https://github.com/letta-ai/letta/commit/bb8cc642) (issue #10110) fixed a fast path for counting agents that was also counting hidden ones. Nothing errored. The endpoint returned a perfectly normal-looking number. It was just the wrong number. That is a textbook silent false success, and it doesn't appear in the 0.16.7 release notes.

I don't think anyone hid it. Curated notes are written for users deciding whether to upgrade. "Agent count no longer includes hidden agents" is a small cosmetic fix from that angle, so it gets cut. Somebody writing notes is doing their job by leaving it out. But for my purposes, that editing step is a filter that removes the exact entries I wanted to count, before I ever see them.

## A rough cross-check

To see whether this was more than one anecdote, I ran a crude comparison over the same window. I took a regex for explicit silence language ("silently", "reported success", "without error", "swallowed", and similar) and ran it over two texts: the fix commit bodies, and the release note entries.

After removing a few obvious false positives, about 10% of the fix commit bodies matched. Only 3.2% of the release note entries did. So commit bodies talk about silence roughly three times as often as the notes do.

Before you quote that, here are the caveats, and they're real ones:

- I wrote the regex **after** I'd already seen the data. It's a post-hoc comparison, not a test.
- It only catches people who used the words. Nobody read the matching lines to judge whether each one was actually a silent bug.
- It's one project and one short release window.

So treat that ratio as a hint about direction, not as a finding. It's consistent with the filtering story. It doesn't prove it.

## The lessons I'm taking forward

**1. Pick a sample that wasn't edited with your question in mind.** Release notes are edited for upgrade decisions. My question was about failure modes. Those two goals don't line up, and the editor's goals win, because they made their choices before I arrived. Commit bodies are messier, but nobody was trimming them for an audience.

**2. Measure your coverage before you trust your sample.** The fix-commit-to-notes mapping was the most useful part of the whole exercise. Without it, I would have gotten a set of percentages from the notes and had no idea they described a filtered subset. If you study any curated artifact (changelogs, postmortem summaries, incident dashboards), check first what fraction of the underlying events actually made it in.

**3. Don't oversell your instrument either.** My mapping was keyword-based: match a commit to a note entry by shared terms. When I spot-checked it by hand, only somewhere between about 17% and 30% of the matches were right. That means the mapping can't even give you an upper bound on coverage. A rough tool can tell you something is wrong. It can't tell you how wrong.

**4. "Inconclusive" with a clear reason beats a number you can't defend.** The tempting move was to loosen the rules after the fact until something cleared the bar. I'd written the rules down first specifically so I couldn't do that. What I got instead was more useful: a concrete reason the question can't be answered from this kind of source, and a pointer to where it might be answered.

**5. Don't generalize from one project.** I'm not claiming memory-layer libraries in general under-report silent bugs in their notes. I have one project and one window. What I'm confident about is the method: curated notes aren't a fit for this question.

## What I did next

The follow-up round moved the unit of measurement from release note entries to fix commits, read in full, on a different project and a single release window. I wrote the categories down before judging any entry, used regexes only to narrow down candidates, and checked coverage before freezing anything.

My own reading came back inconclusive again, just barely. I had also written down in advance that a second, blind reader's labels would replace mine on the borderline rows, and that if the gap closed under those labels, I'd record the result as a refutation of the idea I was testing (that status-related fixes are more often silent than other fixes). It did close, so under that pre-registered rule the result is recorded as refuted. I'm not calling that a solid refutation, and the limits matter as much as the label: the blind reader and I agreed on only 4 of 12 rows; the flip came from two rows nobody had flagged as risky; and it depends on the blind reader's looser reading of what counts as a moderately clear silent bug. Under my stricter reading, it would still be inconclusive. That one deserves its own post.

If you maintain a project, here's a small, low-effort request. When you fix a bug where something *looked* fine and wasn't, say so in the commit message. "Count included hidden agents; returned a plausible but wrong number" is one line. It's the line that tells the next person what kind of bug this was, even if it never makes it into the release notes, and it probably shouldn't have to.
