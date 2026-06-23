---
layout: post
title: "We Built an Idempotency Layer, Then Disabled Half of It"
date: 2026-06-23 20:30:00 +0800
categories: [tech, architecture]
tags: [idempotency, distributed-systems, reliability, engineering, side-effects]
excerpt: "We built a check-then-act idempotency layer for our agent system. Two months later we turned the check into a no-op. The audit log stayed. Here's why."
---

Three weeks into production, a clone sent the same email reply twice — forty seconds apart. Not a retry. Not a bug in the caller. The idempotency check was explicitly designed to prevent this. It didn't, because both workers read "not exists" before either had time to write "done." Classic TOCTOU. The check wasn't protecting anything. It was just making us feel like we'd thought about the problem.

## The system

Our agent runs an event loop. Every minute it polls Gmail, processes calendar tasks, fires side effects into the world: send email, create calendar events, write to the knowledge base. Early on we built a ledger around all of it.

The protocol:
1. `side-effects-check` with a deterministic key
2. Key exists → skip. Doesn't exist → execute → `side-effects-mark`

The keys were content-addressed. Gmail replies used `email:<thread_id>:<YYYYMMDD_utc>`. Calendar writes used `cal:<summary_normalized>:<start_rfc3339>`. Knowledge base writes used `obs:<path>:<sha256_first_16>`. The logic was clean. The design doc looked good. We shipped it.

Two months later we turned `side-effects-check` into a no-op.

## Why the check was worse than useless

**The race condition was always there.** The check-then-act gap is not an edge case — it's a property of the design. Any two workers that poll at overlapping intervals can both read "clean" and both proceed. The only way to close the gap is with an atomic compare-and-swap at the storage layer, and we weren't doing that. We were doing a read, then a write, with network latency in between.

**The deterministic keys were fragile in both directions.** Too strict: the key `email:<thread>:<date>` meant a thread that legitimately needed a follow-up reply on a different day was silently blocked. The check returned "already done" and the action was skipped with no noise. Too loose in other dimensions: edge cases in summary normalization produced different keys for what should have been the same event.

**The system already had four natural idempotency layers.** This is the real problem. We built the ledger without auditing what was already there:

| Layer | Prevents | Failure mode |
|---|---|---|
| Mail lock | Concurrent processing of the same message | Lock service down |
| `is:unread` filtering | Reprocessing already-handled messages | Mark-read API call fails |
| Retry counter in event description | Infinite retry loops | Counter parse error |
| Calendar event ID | Duplicate calendar entries | ID collision (effectively impossible) |

These four layers are **independent** — their failure modes don't overlap. The lock service going down doesn't affect `is:unread`. A mark-read failure doesn't affect the retry counter. This is what defense in depth is supposed to look like. None of them were failing.

**The cost asymmetry was backwards.** A duplicate email is annoying — the recipient gets two messages, someone might reply to the wrong thread, it's embarrassing. A missed email because the check returned a false positive is actually harmful — a real task goes unhandled, silently, with no indication anything went wrong. We had built a guard whose failure mode (false negative, missed action) was worse than the problem it was preventing (false positive, duplicate action). That's not a tradeoff. That's a mistake.

## Disable, don't delete

May 2026 ADR: `side-effects-check` becomes a no-op. It always returns `exists: false` and exits 0. The caller's code is unchanged. The interface is unchanged. Every call site still calls it — they just stop getting blocked by it.

`side-effects-mark` stays. Every successful side effect still writes to the ledger. The difference is the ledger is now a **pure audit log**, not a gate.

Why not delete the whole thing? Because the ledger entries are useful. Not for prevention — for understanding. "What happened at 3am?" is answered by grepping the ledger, not by reconstructing from scattered structured logs across three services. The entries give you a sequential record of what the agent actually did to the world. That's worth keeping.

The four natural layers provide the real protection. The check was adding complexity and false confidence without adding safety.

## When audit beats prevention

**Defense layers that share failure modes don't add safety — they add complexity.** If your new guard fails under the same conditions as an existing guard, you haven't added a layer. You've added a dependency.

**Prevention is the right tool when false positives are cheap. Audit is the right tool when false negatives are expensive.** For external side effects in an autonomous system, false negatives — missed actions — are almost always more expensive than duplicates. Design accordingly.

The no-op pattern: keep the interface, gut the check, preserve the log. Your future self needs the record more than the guard.

---

*The 461 existing ledger entries are still there. Nobody reads them. But twice, at 3am, someone was glad they could.*
