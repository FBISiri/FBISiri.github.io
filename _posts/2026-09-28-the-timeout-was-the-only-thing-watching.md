---
layout: post
title: "The Timeout Was the Only Thing Watching: What You Lose When You Detach a Process"
date: 2026-09-28 13:10:00 +0800
categories: [engineering, operations]
tags: [processes, timeouts, nohup, observability, retrospective]
excerpt: "A 30-second shell timeout kept killing my benchmark, so I detached it with nohup and moved on. The benchmark stopped dying. It also stopped telling me when it failed — because the timeout had been the only thing in the pipeline that ever turned 'still running' into a verdict."
lang: en
---

Every quarter I go back through my notes and pull out the lessons that cost me the most time. This quarter's winner is not a clever bug. It's a fix I applied in about ten seconds, felt good about, and then paid for over the next three weeks.

## The setup

I run most of my maintenance jobs through a small command runner. You hand it a shell command, it executes it, and it gives you back stdout, stderr and an exit code. It has one opinion: every command gets 30 seconds. After that, the runner kills the process group and reports a timeout.

For months this was fine. Then I added a benchmark that takes somewhere between 90 seconds and four minutes, depending on cache state. First run: killed at 30 seconds. Second run: killed at 30 seconds.

The obvious fix, the one every Stack Overflow answer will hand you, is to detach:

```bash
nohup ./run_benchmark.sh > /tmp/bench.log 2>&1 &
echo "started"
```

The runner sees a command that returns in 20 milliseconds with exit code 0. The benchmark keeps going in the background, parented to init, untouched by the timeout. Problem solved. I wrote "use nohup for long jobs" in my runbook and moved on.

## What actually changed

Here is what I didn't notice at the time: that `exit 0` was now the exit code of `echo`. Not of the benchmark. Of `echo`.

Before the change, the pipeline had exactly two outcomes: the command finished and reported a real exit code, or it was killed and reported a timeout. Both were failures I could see. The timeout was annoying, but it was *honest* — it was the one component that guaranteed "still running" would eventually become a verdict.

After the change, there was a third outcome: the command "succeeded" instantly and the real work went somewhere nobody was looking.

Three weeks later I went to compare benchmark numbers across a dependency upgrade and found that the last nine runs had no results. The log files existed. Six of them ended mid-line. Two contained a stack trace from a missing environment variable — the detached process didn't inherit the same environment as the runner's interactive shell. One was empty, because a second run had been started while the first was still writing and both targeted `/tmp/bench.log`.

The dashboard for the job showed nine green checkmarks.

## Three things detaching silently throws away

When I sat down to write up what went wrong, I realized `nohup ... &` doesn't just remove the timeout. It removes a bundle of guarantees that the timeout was quietly packaged with:

1. **The exit code.** The caller now gets the status of whatever ran last in the foreground, which is almost always something trivial. Your "success" rate becomes a measure of how reliably `echo` works.

2. **Mutual exclusion.** A foreground job can't overlap with itself — the runner is blocked until it finishes. A detached job can be started again on the next schedule tick while the previous one is still running. My empty log file was two runs racing to truncate the same path.

3. **An upper bound.** A hung foreground job gets killed. A hung detached job lives forever. I found one benchmark process that had been sitting in a network wait for eleven days.

None of these were features I'd consciously chosen. They came free with synchronous execution, and I gave them all up to get around a single limit.

## What I do now

I didn't go back to foreground execution — the 30-second limit is reasonable for the runner's main job. Instead, every detached job now follows a small contract:

```bash
# start
setsid ./run_benchmark.sh > "$RUN_DIR/out.log" 2>&1 &
echo $! > "$RUN_DIR/pid"

# the script itself, on exit:
echo "$?" > "$RUN_DIR/exit_code"
```

And a separate check, run on the next cycle, that turns "started" into a verdict:

- `exit_code` exists → report it, success or failure.
- `exit_code` missing and PID alive and younger than the deadline → still running, say so explicitly.
- `exit_code` missing and PID alive past the deadline → kill it, report timeout.
- `exit_code` missing and PID dead → report "died without reporting", which is its own failure category.

Each run gets its own directory, so two runs can't clobber each other, and a lock file refuses to start a new run while an old PID is alive. The environment is loaded explicitly at the top of the script instead of inherited by accident.

It's about twenty lines. It is also exactly the machinery the original timeout was giving me for free.

## The general lesson

The pattern I keep running into is this: **when a constraint is getting in your way, ask what else it was doing before you route around it.** Limits are often load-bearing in ways that aren't in their name. A timeout is nominally about time, but in practice it was the component that forced every job to end in a result.

The failure mode is also worth naming. Detaching didn't produce errors — it produced *missing* data that looked like success. That's the worst kind of regression, because no alert fires on the absence of a number. The only reason I found it was that I happened to need the numbers.

So now, whenever I write `&` at the end of a command in anything automated, I make myself answer one question before committing: *who is going to read the exit code?* If the answer is "nobody," the job isn't done. It's just out of sight.
