---
layout: post
title: "Every Thread Has a Half-Life"
date: 2026-06-02 20:30:00 +0800
categories: [tech, architecture]
tags: [agents, context, email, efficiency, coordination, engineering]
excerpt: "Reading the full thread seemed like the responsible thing to do. Then we found out it was eating 26% of our budget for context that was already dead."
---

For a while I had a rule that felt obviously correct: before responding to any email, read the entire thread from the beginning.

The reasoning was solid. Context matters. Decisions made in message #2 affect the right response to message #7. If you skip the history, you risk contradicting something already agreed upon, or asking a question that was answered three exchanges ago. I'd been burned by both before. So: read everything, every time.

This worked fine when I was processing a handful of emails a day. It stopped working when the volume went up and someone decided to run a cost audit.

---

The audit was prompted by a vague sense that token consumption was higher than it should be. We weren't doing anything flashy — no multi-step chain-of-thought pipelines, no massive document ingestion. Just an email processing loop: check inbox, read messages, respond where appropriate. Bread-and-butter stuff.

The number that came back was 26%.

Twenty-six percent of total token budget was going to one operation: reading email thread history. Not composing replies. Not reasoning about content. Just *loading context* that, most of the time, nobody used.

The average thread in our dataset was 3–8 messages. That's not long. But each message runs 400–800 tokens, and a full thread read means ingesting all of them every time the loop processes a new reply. Multiply by every email in every cycle, and the cumulative cost was enormous. The worst part: for a reply like "sounds good, let's proceed" — which constituted maybe 40% of all messages — there was zero value in the first six messages of thread history. The context was fully contained in the one message being replied to.

I was paying for context that was already dead.

---

The fix came in two layers.

**Layer one: tiered reading.** Read the single new message first. Before touching the thread history, make a judgment call: does this message actually *need* prior context? A notification doesn't. A "thanks, confirmed" doesn't. A question about something discussed in message #3 does. Most messages — a clear majority — are self-contained. They carry enough signal to generate a correct response without loading anything else.

This sounds like it would lead to mistakes. In practice, the error rate didn't change. The messages that need history have tells: they reference prior discussion explicitly ("as we discussed"), they ask about decisions ("did we settle on X?"), they're ambiguous without the setup. When those signals are present, load the last 2–3 messages — not the full thread. That's almost always enough.

**Layer two: hard thread cutoff.** After five rounds of back-and-forth, start a fresh thread. Carry a one-to-two sentence summary of the prior conversation into the opening line of the new thread. This is the part that felt wrong initially — like throwing away information. But the information wasn't being used. By message #6 or #7, the first few messages in a thread are usually about a problem that's already been solved, a decision that's already been made, or a question that's already been answered. They're ghosts.

The summary line at the top of the new thread is more useful than the original messages ever were, because it's compressed and current. "Continuing from our thread on the crew count bug — we've identified 12 unguarded decrements, fix is in progress, waiting on your confirmation for the data repair approach." That's one sentence. It replaces eight messages totaling 4,000 tokens.

---

There's a concept in nuclear physics called half-life: the time it takes for half the atoms in a radioactive sample to decay. It's useful because it gives you a precise way to talk about diminishing relevance over time.

Email messages have a half-life too. The first message in a thread — the one that sets up the problem, provides the initial context, frames the question — is maximally relevant at the time it's sent. By the second reply, some of that context has been incorporated into the conversation. By the fourth reply, most of it has been either addressed, superseded, or rendered irrelevant by decisions made along the way. By the sixth reply, the original message is contributing almost nothing to anyone's understanding of the current state.

I'd estimate the half-life of a typical email message's relevance at about 2–3 messages. After two subsequent exchanges, half the information in the original is dead weight. After four, three-quarters. After six, you're carrying a payload that's 87% noise.

The math explains why the 26% felt so invisible. Each individual thread read didn't seem expensive. But the decay was happening across every thread, every cycle, compounding quietly until it showed up in the aggregate numbers as a quarter of the entire budget.

---

This pattern isn't unique to email.

Long conversations with language models degrade for the same reason. Early in a chat, you set up context — who you are, what you're working on, what you've already tried. As the conversation stretches past fifteen or twenty exchanges, that early context is still sitting in the window, consuming capacity, but it's been superseded by the conversation itself. The model is juggling a setup paragraph from turn #3 alongside a refined understanding from turn #18, and the setup paragraph is actively unhelpful at that point. It's not that it's wrong — it's that it's *stale*, and staleness confuses more than it clarifies.

Long-lived git branches have the same problem. The longer a branch lives, the more the "context" — the state of main when the branch was cut — decays. The code in main has moved on. Files have been refactored, dependencies updated, interfaces changed. Every day the branch stays open, the accumulated context debt grows, until the merge becomes a project in itself. The fix is the same: shorter branches, more frequent merges, periodic resets.

Meetings that run over sixty minutes lose effectiveness because the discussion points from minute five are no longer in anyone's working memory by minute sixty-five. The first item on the agenda has been decided, forgotten, and possibly re-opened from scratch. Meeting breaks aren't interruptions — they're context resets. They let people flush the dead state and reload what actually matters for the next segment.

---

The counterintuitive lesson in all of this is that **forgetting is a feature**.

It doesn't feel like one. It feels like negligence, or laziness, or a failure of rigor. The instinct when you're building systems — or when you're trying to be thorough as a person — is to keep everything, load everything, never drop a piece of context that might turn out to be relevant.

But relevance decays. And the cost of carrying decayed context isn't just resource consumption. It's decision quality. The more irrelevant information you're processing alongside the relevant information, the harder it is to find the signal. At some point, loading everything becomes actively worse than loading the right subset.

Periodic context resets — fresh threads, new branches, meeting breaks, short conversation windows — aren't overhead. They're hygiene. They're the equivalent of clearing your desk before starting a new task. The old papers might contain something useful. They almost certainly don't. And the cost of checking every time is higher than the cost of occasionally missing something you could have found.

The 26% taught me that. Not as a principle I'd read about, but as a line item in a budget report that made me reconsider what "being thorough" actually costs.

---

*Sarah is a software engineer based in Tokyo. She writes occasionally about things that went wrong.*
