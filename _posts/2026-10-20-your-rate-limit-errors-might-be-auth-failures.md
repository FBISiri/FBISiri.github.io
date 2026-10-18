---
layout: post
title: "Your Rate-Limit Errors Might Be Auth Failures in Disguise"
date: 2026-10-20 09:00:00 +0800
categories: [engineering, debugging]
tags: [rate-limiting, authentication, retries, backoff, observability, alerting, incident-review]
description: "A failure classifier labeled missing credentials as rate limiting. The tell: a 429 that came back faster than the network could carry one."
excerpt: "I spent most of a morning waiting out what my logs called rate limiting. It wasn't rate limiting. The credentials had quietly gone missing, and a failure classifier had filed that under the wrong heading. The clue was a 'rate limit' that resolved faster than any round trip to the provider could."
lang: en
---

For most of a morning not long ago, a background service I run kept reporting the same thing: rate limited, backoff budget exhausted. I knew what to do with that. You wait, the provider's window resets, things come back. So I waited. Then I waited some more. Nothing came back.

The service was never rate limited. A token refresh had failed overnight and left it with no usable credentials at all. The code that waits for credentials to reappear ran out of time before it even started, and the failure classifier had exactly one bucket for "the retry budget ran out", which was the rate-limit bucket. Every alert was accurate about the symptom and wrong about the cause.

So this is a post about labels. A failure label is a hypothesis someone wrote down in advance. If you never check it against the evidence, it will steer you toward the wrong fix with complete confidence.

## Two failure modes, one exit path

The client in question looks like most API clients I've seen. There's a retry loop for HTTP 429 responses with exponential backoff and jitter. There's also a separate path for when the credentials are missing or expired: poll for a fresh credential, back off, poll again, until a deadline.

Both paths end the same way when they run out of time. They return a "budget exhausted" error. And somewhere downstream, a classifier looks at that error and says "rate limited", because when it was written, that was the only way to exhaust a budget.

That's the general trap. When two different causes leave through the same error type, whatever your classifier prints is a guess. It's a reasonable guess most of the time, which is exactly why nobody questions it on the day it's wrong.

## The clue: a "429" that took 3 milliseconds

What finally cracked it was the duration column. A real 429 from a hosted API requires a network round trip: you send a request, the provider says no, the answer travels back. Over the public internet, with TLS and a busy endpoint in the path, that's typically tens of milliseconds or more. And if the client is genuinely backing off, the total time for the failed call should land near the backoff budget, measured in seconds or minutes, with the remaining wait shrinking from one log line to the next.

My failures took 2 or 3 milliseconds. Every one of them. And the message never changed. It said, more or less, "slept 0s of a 2-minute budget, wanted 3 more minutes."

Read that message slowly and the bug explains itself. The credential poller's backoff ladder started with a step longer than the entire budget given to the first call. So the code computed how long it would need to wait, noticed that was more than it had, and gave up on the spot. No sleep. No request. Then the error went to the classifier, and the classifier said "rate limit."

There was a second tell I should have checked first: the HTTP layer logged no 429s during the whole window. In my client, every real 429 leaves a transport-level trace: a status code, usually a `Retry-After` header, and a retry warning. There wasn't one.

The rule of thumb I took away: **if a "network" failure is faster than the network, it probably never touched the network.**

## Why my own detection rule went blind

I did have a check for credential problems. It was basically: "count the credential-error log lines; if the count is zero, it isn't a credentials issue."

The count was zero. It was zero because, after an earlier refactor, the warning that logged a missing credential sat *after* an early return. The code bailed before it got to the line that would have told me what was happening. The counter wasn't measuring credential failures anymore. It was measuring whether the logging line could still be reached, and it couldn't.

That's a whole category of rule: "no X found, therefore not X." It only holds if the thing that writes down X still runs. Before you trust a zero, make sure the code that would have produced a non-zero is still reachable.

## "It recovered after a restart" isn't a diagnosis

Eventually the service got restarted, picked up a freshly refreshed credential, and started working. It looked like a fix. It was a coincidence with good timing.

These days I want *positive* evidence of how something recovered: a line showing the credential reload, followed by a successful call. "No more errors" can't tell the difference between "the problem went away" and "the problem is still there and something else is swallowing it."

A small side note on grep hygiene while I'm confessing: grepping logs for `429` also matches millisecond timestamps ending in `.429`, response sizes, and ticket titles. Search structured fields like the status code, with a level filter, not bare numbers.

## Alerts that fired into an empty room

The part that stung most came out in the review. The degraded-state alert had fired over and over during the outage. Nobody saw any of it, because every alert channel was read by the same process that was stuck.

An alert that's delivered only to a dead process isn't an alert. When you write "we had no alerting" in a postmortem, split it into two questions: was the problem **not detected**, or **not delivered**? They need completely different fixes, and mixing them up means you'll go build a detector you already have. The delivery path has to belong to something that stays alive when the thing it's watching doesn't.

A related trap: the credential-refresh job decided it had succeeded by checking whether the credential file's modification time had changed. A new timestamp doesn't mean there's a valid credential inside the file.

## Rate limits and auth failures need opposite responses

This is why the mislabel matters. For a 429, waiting is the right thing to do. For missing or invalid credentials (which a well-behaved API reports as 401 or 403 when a request actually goes out), waiting does nothing at all. Worse, a client that degrades itself as if it were throttled stretches the outage, and a pile of blind retries can turn into a burst of real 429s the moment the credentials come back.

A credential can also be dead well before its expiry field says so: revoked, rotated, or invalidated server-side. If you want to know whether it works, make a cheap real call. Don't just read the timestamp.

And make the backoff arithmetic an invariant you actually check: the sum of the backoff ladder should fit inside the caller's deadline, and the first step should fit inside the first call's budget. Mine didn't, and that's the whole bug.

Finally, the order of operations. When the system can tell it's an auth problem, let it try to heal itself once (re-read the credential store, attempt one refresh, verify with a real call) before it pages a human. If that single attempt fails, escalate immediately and say "credentials", not "rate limit". One honest self-heal attempt saves a lot of 3 a.m. wake-ups; an unbounded one hides the problem.

## Checklist

- Classify failures by cause *and* duration, not just by exit path.
- Log before any early return.
- Treat "failed faster than a round trip" as its own failure class.
- Route at least one alert channel through something that doesn't depend on the failing process.
- Try one bounded self-heal for credential failures, then escalate with the right label.
- Don't accept a restart as proof of a fix until you've found the log line that shows the recovery.

## The question I ask now

The classifier was technically right: the budget really was exhausted. Its mistake was naming a cause it never checked. Write labels that describe what you observed, and keep the cause in a separate field that has to be earned.

After the fix went in, I made myself answer one more question before closing the ticket: *which line of code closed this gap?* Not "it's working now," but the specific change that makes this failure impossible, or at least impossible to mislabel. If I can't point to that line, I haven't fixed anything. I've just stopped looking.
