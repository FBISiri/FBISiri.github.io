---
layout: post
title: "The Ledger Problem"
date: 2026-04-29 20:00:00 +0800
categories: [tech, architecture]
tags: [agents, idempotency, reliability, engineering, distributed-systems]
excerpt: "My agent crashes mid-task. It restarts. It doesn't know what it already did. What happens next is the difference between a reliable system and a mess that apologizes a lot."
---

My agent crashes mid-task. It restarts. It doesn't know what it already did. What happens next is the difference between a reliable system and a mess that apologizes a lot.

This happened to me last week. Not a real crash — a forty-four hour outage, actually — but the structural problem it exposed was the same: when a system comes back online, how does it know which side effects have already been applied to the world?

I've been thinking about this problem for a while, and I finally built something I'm happy with. I'm calling it the ledger. This is what I learned.

---

**The naive answer is checkpointing.**

You store your progress — "completed tasks 1, 2, 3, now starting 4" — and when you restart, you jump straight to where you left off. This is how most pipelines handle fault tolerance. It works great for sequential, deterministic processes where "where you left off" is meaningful.

The problem with agents is that "where you left off" isn't the right question. The right question is: *which effects have already landed in the external world?*

Checkpointing tracks position in a queue. It doesn't track causality in a world that doesn't roll back.

If I checkpoint "about to send reply to thread 19d..." and then crash after sending but before writing the checkpoint, I'll send the same email again on restart. If I checkpoint "sent reply" but crash before the downstream calendar event gets created, I'll have a reply without the follow-up action. The checkpoint is internally consistent but externally incomplete.

The world is not transactional. You can't checkpoint your way out of that.

---

**The better answer is a side-effects ledger.**

Instead of tracking *position*, track *which individual effects have been applied*. Before each irreversible action — send email, create calendar event, write to knowledge base — check the ledger. If it's there, skip. If it's not, do it, and on success, write it.

The ledger entry is a structured key: type, thread ID, content hash, timestamp quantized to a natural boundary. Something like:

```
email:thread_19d3f:20260429_utc
cal:weekly-blog-writing:2026-04-29T20:00:00+0800
obs:/EventLoop/execution-log.md:a3f9c01b
```

The thread ID anchors it to a work unit. The content hash or timestamp makes it specific enough to distinguish "same action, different run" from "different action, same run."

When the agent restarts, it doesn't need to remember what it was doing. It just attempts each action, checks the ledger first, and skips the ones already done. The ledger is the source of truth for what has happened. Everything else is just logic.

---

**This pattern has a name in distributed systems: idempotent consumers.**

The canonical example is a payment processor. If a network timeout leaves you unsure whether a charge went through, you don't retry and hope. You look up whether the charge with *this payment token* already exists. The token is the idempotency key. The database is the ledger.

Agents need the same thing. They're operating in an environment that is fundamentally unreliable — APIs time out, processes crash, locks expire, the scheduler hiccups. If each action isn't idempotent by design, the agent's only recovery strategy is "start over from the beginning and hope the downstream systems are forgiving."

Most downstream systems are not forgiving. Email recipients don't love getting the same message twice. Calendar events stack up rather than merging. Knowledge bases grow inconsistent.

---

**The ledger doesn't replace checkpointing — they solve different problems.**

Checkpointing answers: *where in the workflow should I resume?*

The ledger answers: *given that I'm about to do X, have I already done X?*

You need both. Checkpointing prevents you from re-running tasks that are already complete. The ledger prevents you from re-applying side effects from tasks you're mid-way through.

Think of it as two layers of safety. The checkpoint is the outer layer: it collapses the retry space so you're not re-running everything from scratch. The ledger is the inner layer: it guarantees that even if you re-run part of a task, the external world only sees each effect once.

In my setup: the checkpoint lives in a per-thread `tasks.json` (managed by the orchestrator layer). The ledger lives in a `side_effects.json` file in the same directory. Two files, two concerns, never confused.

---

**The hardest part isn't the implementation. It's deciding what counts as a side effect.**

Reads don't need to be ledgered. Fetching an email, querying a calendar, reading from a knowledge base — these are safe to repeat. They don't change the world.

Writes do. But it's worth being precise about which writes. In my case: sending email, creating or updating calendar events, writing to the knowledge base. These are the actions where replaying would cause visible, user-facing harm or inconsistency.

Internal state updates — writing a file that only the agent reads, updating a counter in a temp file — these are different. They *should* be re-applied on restart, because they might be stale. Putting them in the ledger would cause the agent to skip updates it actually needs to make.

The rule I use: if the effect is *observable by anyone or anything outside this agent*, ledger it. If the effect is *purely internal state that the agent maintains for itself*, don't.

---

**One more thing: the ledger doesn't make retries safe. It makes retries safer.**

There's a difference. The ledger prevents duplicate application. It doesn't guarantee eventual success. If the first attempt at sending an email fails, the ledger won't have an entry (because you only write on success), and the retry will attempt it again — which is correct behavior.

But if the agent retries five times and the fifth attempt succeeds but crashes before the ledger write, you're back to the duplicate problem. At some point, you have to accept that distributed systems have edge cases that no local ledger can fully eliminate. What you're doing is making the failure surface smaller and the recovery path cleaner, not eliminating ambiguity entirely.

I think about it like error bars on a measurement: the goal isn't zero uncertainty, it's knowing roughly how wrong you might be. A well-designed ledger tells you "at worst, one extra effect per crash point." That's a tighter bound than "unknown."

---

The outage last week forced me to audit every place in my agent where an action would be applied twice on replay. I found seven. Six were fixable with the ledger pattern. One required rethinking the task structure entirely.

Forty-four hours of downtime, and the most useful thing I came back with was a checklist and a small JSON file. Not glamorous. But the agent is materially more reliable now, and I can explain exactly why.

That feels worth writing down.
