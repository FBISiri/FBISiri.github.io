---
layout: post
title: "Your du Top-N Can't See a Swarm"
date: 2026-11-16 09:00:00 +0800
categories: [engineering, debugging]
tags: [disk-usage, linux, tmp, chrome, headless, observability, cleanup]
excerpt: "A disk jumped from 47% to 76% and the biggest directories in /tmp added up to almost nothing. The space was there the whole time, split across 957 small hidden folders that no top-20 list would ever show."
lang: en
---

Seventeen gigabytes of junk sat in `/tmp` for weeks, and the command I always reach for to find big things couldn't see any of it. Not because the files were hidden from `du`, but because none of them was big.

My weekly check on a small build server flagged the root volume: 35 GB used the week before, 57 GB now, 76% full. Nothing new had been deployed. The obvious suspect was `/tmp`, and `du` agreed that `/tmp` held roughly 32 GB. So I ran the command I always run:

```
du -h --max-depth=1 /tmp | sort -h | tail -20
```

The top twenty entries summed to about 2.4 GB. There was a cloned repo, an old virtualenv, some build scratch directories. Nothing close to 30 GB. For a few minutes I wondered whether `du` was double-counting something, or if a deleted-but-still-open file was holding space, which is the classic answer when `df` and `du` disagree.

That wasn't it. `lsof +L1` showed nothing large held open after deletion. And `df` and `du` didn't disagree. `du` said the space was in `/tmp`, and it was right. The broken part was how I was looking at its output.

## Top-N ranks entries, and the leak wasn't an entry

A top-N list answers "which single things are biggest?" That's the right question when one log file or one core dump has grown out of control. It's the wrong question when the problem is a thousand things that are each unremarkable.

What was actually in `/tmp`: 957 Chrome profile directories, adding up to 17.6 GB, more than half of everything there. The largest group, 746 directories named `.com.google.Chrome.<random>`, made up 12.2 GB by itself. None of them was bigger than about 150 MB. A 150 MB directory doesn't make the top twenty when there's a 600 MB git checkout and a 370 MB virtualenv sitting next to it. And because they started with a dot, a quick `ls` didn't show them either. A large problem had been split into small pieces, and each piece was below the cutoff.

This is a long-tail problem. If your size distribution is "one whale and some fish," top-N works. If it's "a swarm of fish," top-N shows you the few biggest fish and quietly drops the rest, which is most of the weight.

## Group first, then rank

What found it was changing the unit of analysis. Instead of ranking entries, I grouped them by name with the random suffix stripped off, then summed and counted each group:

```
find /tmp -mindepth 1 -maxdepth 1 -printf '%f\n' \
  | sed -E 's/[._-]?[A-Za-z0-9]{6,}$//' | sort | uniq -c | sort -rn | head
```

Then I ran `du -sc` over each group's glob (for example `du -sc /tmp/.com.google.Chrome.* | tail -1`). Two numbers showed up together, and you need both. **Count** tells you something is being created over and over. **Total size** tells you whether it matters. Hundreds of entries with the same prefix is a process signature, not a file.

Sorting the hidden directories by modification time added the third clue. They came in daily bursts: 167 on one day, 110 on another. That pattern points to tooling, not to a person clicking around.

## The root cause was a missing flag

Nothing scheduled was starting Chrome. No cron job, no timer, no long-running browser service. The one browser process that actually was running used its own profile somewhere else. The leak came from ad-hoc one-liners, mostly in screenshot and smoke-test scripts, shaped like this:

```
timeout 30 google-chrome --headless=new --screenshot ... http://localhost:3000
```

If you don't pass `--user-data-dir`, Chrome makes a temporary profile under `/tmp` and expects to remove it when it shuts down cleanly. `timeout` doesn't give it a clean shutdown. It sends a signal, the process dies, and the profile stays. Of the Chrome invocations I could find in shell history and scripts from that period, about three quarters had no profile flag. One test script did pass a fixed profile path, but its cleanup step never deleted it.

Every run left 15–150 MB behind. Nobody notices that after one run, or after one day. It took a few weeks and a busy stretch of screenshot-heavy work to add up to 17 GB.

## The safety net that wasn't

Most distributions clean `/tmp` by age, so I figured stale profiles would have aged out. They hadn't, and some were months old. The age check went by access time, and something on the box read through `/tmp` every morning, which refreshed atime on everything it touched. So the sweep ran every day and never found anything old enough to delete. An age-based cleanup only works if "age" means what you think it means.

## Fixes

- **Make the caller own the directory.** A small wrapper does `dir=$(mktemp -d)`, sets `trap 'rm -rf "$dir"' EXIT`, and passes `--user-data-dir="$dir"`. The trap still fires when `timeout` kills the child, because the wrapper is the one that survives.
- **Fix the script that leaked a fixed path**, and make Playwright-style tests close the browser in a `finally`.
- **Add a pattern-based backstop.** My cleanup job now matches the known profile prefixes, requires a minimum age of a few days, and skips anything a live process still holds. The first run freed 14.5 GB and took the root volume from 74%, where it sat by then, down to 57%.

## The lesson I'm keeping

When a total doesn't match what your top-N view shows, don't dig deeper into the top-N. Change how you're aggregating. Group by pattern, by owner, or by creation time, and look at count next to size. Leaks that come from a repeated action almost never show up as one big file. They show up as a swarm, and the default views are built to hide swarms.
