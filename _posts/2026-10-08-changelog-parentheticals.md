---
layout: post
title: "Who's in the Parentheses? Reading a CHANGELOG After the Issue Tracker Moved"
date: 2026-10-08 09:56:35 +0800
categories: [engineering, methodology]
tags: [changelog, open-source, attribution, issue-trackers, git-archaeology]
excerpt: "I set out to say who CHANGELOG credits usually name. The claim didn't survive contact with the source, but the way it failed taught me how to read release notes: notation first, names second, and never trust a #N that crossed a tracker migration."
lang: en
---

I went into this one wanting a tidy claim. A couple of weeks ago I was reading fix entries in an open-source memory library, [doobidoo/mcp-memory-service](https://github.com/doobidoo/mcp-memory-service), and I noticed that a lot of entries end in a parenthetical with a name in it. My working theory was that those names mostly credit whoever *reported* the problem, not whoever fixed it. That would have been a nice, quotable finding.

It didn't hold up. When I checked it against commits and the project's own docs, the claim fell apart as stated, and I'm not going to swap in a different headline figure worked out after the fact. What I do have is a better sense of *how* to read these parentheticals, plus one genuinely sneaky failure mode I hadn't seen written up anywhere. So this is a post about reading, not about a scoreboard.

Before anything else: the reason I could check any of this is that the maintainer had already done the hard part. More on that below.

## Notation is the first variable, not the name

The first thing I got wrong was treating "a name in parentheses" as a single kind of thing. In this one CHANGELOG I found at least four distinct shapes:

1. **`reported by @X`** — explicitly labelled. X reported the issue. Easy.
2. **A bare `@X` next to a fix** (from the stretch when the project was hosted on Codeberg) — here X is the author of the change.
3. **`(#PR, handle, closes #issue)`** (GitHub era, up to v11.14) — the first number is the pull request, the handle is its author, and the number after `closes` is the issue. The separator isn't even consistent: sometimes a comma, sometimes a semicolon, as in `(#1291, massimiliano1991; closes #1289)`.
4. **`(#N, handle)`** with no `@` and no `closes` (the v11.15 section) — and this is where it gets interesting. Here `#N` is the *issue*, not the PR. When that section wants to mention a PR it says so explicitly, like `(#1099, PR #1367)`. And the handle's role isn't fixed. In one line it's the person who reported the issue. In another, `(#1201, ghulands)`, it's the author of the earlier code the fix built on. In another, it's the maintainer noting his own report.

So the same visual pattern — number, comma, name — means different things depending on which part of the file you're in. If you average across all of them you get a number that describes nothing.

The rule I ended up with: **classify the notation before you attribute anyone.** If there's a label, trust the label. If there isn't, figure out which era and which format you're looking at first, and only then decide what the name means.

There's also a small, embarrassing lesson in here. My first pass used a regex that required an `@`. That silently selected only one era's notation and skipped the GitHub-era credits entirely, because those don't use `@`. The measurement I had planned only ever saw the Codeberg-era entries, whose numbers I couldn't resolve at the time, so it came back inconclusive. The instrument was looking at the wrong population. A pattern match that quietly picks one format is a sampling decision, whether you meant it to be or not.

## The tracker moved, and the numbers came along

Here's the part I think is worth more than the original question.

For a while, this project tracked issues on Codeberg, then moved back to GitHub. The CHANGELOG says so plainly near the top: numbers below #341 in the v10.71.0 through v11.10.0 entries are Codeberg numbers.

But the CHANGELOG is rendered on GitHub, and GitHub auto-links every `#N` to whatever GitHub object currently has that number. So take the CHANGELOG's #214. In context it's a Codeberg pull request fixing Milvus behaviour. Click it on GitHub and you land on GitHub's #214, which is something else entirely.

The key point: **it doesn't 404.** You get a real page, with a real title, a real author and real comments. Nothing tells you that you're looking at the wrong object. A broken link at least announces itself. A link that resolves to a plausible neighbour doesn't.

Who gets hurt by this? Anyone doing attribution from rendered release notes. Anyone doing security archaeology ("which change fixed this, and who wrote it?"). Anyone writing a blog post with a confident claim about who CHANGELOGs credit. I was on track to be all three.

## How I recovered the right objects

The good news is that the project had already solved most of this. The repo ships a mapping file, `docs/codeberg-issue-map.md`, added in commit `07971a6` (GitHub PR #1126, September 2026). For each Codeberg number it records whether it was an issue or a pull request, and its title. The titles line up with the CHANGELOG entries, which is how I confirmed that reading those numbers as Codeberg numbers was right in the first place.

The map doesn't record authors, so I went to git history for that:

- **Codeberg-era squash commits** carry author emails of the form `<handle>@noreply.codeberg.org`, which gives you the author directly.
- **Merge commits**: look at the commits on the merged branch (the range between the merge's two parents) and take their author.
- **Reporters aren't in git at all.** The only trace is prose in commit bodies, like "Reported by @X in #N". That's secondhand, so I kept it in a separate column from commit authorship.

I want to be clear about the credit here. A maintainer sitting down to write a migration map is the single most useful thing anyone did for future readers of this project, me included. If you ever move trackers, write the map. It costs an afternoon, and without it every link in the migration window is quietly wrong.

## A reading checklist

If you're pulling facts out of someone else's CHANGELOG, here's what I'd do now:

- **Read the preamble first.** Look for tracker-migration notes before you trust any `#N`.
- **Classify each parenthetical's notation** before reading the name. Don't apply one era's convention to another era's text.
- **For squash commits, only the trailing `(#N)` in the subject is the PR.** An earlier `(#N)` in the same subject is usually the issue it closes. `git log --grep='#N'` will match both, so check which position you hit.
- **If the project moved trackers, find the mapping file.** If there isn't one, treat numbers inside the migration window as unresolved. Don't follow the auto-link.
- **Keep firsthand and secondhand evidence apart.** A commit author is firsthand. "Reported by" in prose is someone's account of it.
- **Treat anything you count from a CHANGELOG as a floor, not a measurement.** A CHANGELOG is a lossy compression of the commit history, so whatever you're counting, some of it got squeezed out before it reached the release notes.

## What I'm not claiming

I'm not putting a number on how often parentheticals credit reporters versus authors. My planned measurement came back inconclusive because of the instrument problem above, and swapping in a figure I computed after the fact would be exactly the move I'd criticise in someone else's post.

These conventions also come from one repository across a few release lines. Other projects have their own habits, and some will flip the conventions entirely. The method carries over; the specific mapping doesn't.

The open thread: a proper re-test would match both notations from the start and split everything by tracker era before counting anything. Maybe that's a future post.

## Closing

Release notes are written for people skimming a release, not for attribution audits, and that's fine. It just means the reader has to do some classification work the format doesn't do for you.

Read the parentheses as a format first and a name second.
