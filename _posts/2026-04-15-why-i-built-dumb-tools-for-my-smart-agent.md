---
layout: post
title: "Why I Built Dumb Tools for My Smart Agent"
date: 2026-04-15 22:00:00 +0800
categories: [tech, infrastructure]
tags: [go, agents, performance, architecture, tools]
excerpt: "When your AI agent runs every minute, you quickly learn the difference between tasks that need a brain and tasks that just need to be fast."
---

There's a moment in every project where you catch yourself doing something inefficient and think, *I've been doing this wrong.* Not catastrophically wrong — just quietly, expensively wrong, in a way that compounds over time.

For me, that moment came at about 2am on a Tuesday in my apartment in Shimokitazawa, watching my agent loop churn through a sequence of tool calls that, in retrospect, had no business involving a language model at all.

---

When I first built out the event loop for my AI agent — the system that runs roughly every minute, checks in on emails and calendar, makes small decisions, keeps things moving — I routed everything through MCP. Model Context Protocol. The idea was flexibility: one interface, everything goes through tool calls, the model decides what to fetch, what to do, what to skip. It felt elegant. Unified.

And it *was* elegant, for a while. The problem is that elegance at the architecture level sometimes hides waste at the execution level. Every single cycle, the agent was calling out to fetch unread emails, read a config file, pull calendar events — and it was doing all of that through the same high-token, full-context pipeline it uses for actual decisions. The plumbing was getting treated like reasoning. A round-trip to check if there were new emails had the same overhead as a round-trip to decide how to respond to one.

The tokens add up. The latency adds up. And eventually you start to feel it — not in any single run, but in the texture of the thing. It felt sluggish in a way I couldn't immediately put my finger on.

---

So I did something that felt almost old-fashioned: I wrote Go.

Not a framework. Not an abstraction layer. Just a small collection of CLI binaries that do one thing each, fast, with no ceremony. I called the package `siri-tools`, and right now it has five commands: `read-file`, `gmail-fetch-unread`, `gmail-mark-read`, `event-loop-prep`, and `calendar-get-event`.

The first four are almost embarrassingly simple. `read-file` reads a file and returns its content — with some compression baked in, because I found that a lot of what gets passed back is whitespace-padded or redundantly structured. `gmail-fetch-unread` hits the Gmail API and returns unread messages in a tight, pre-shaped format. `gmail-mark-read` marks a list of message IDs as read. `calendar-get-event` does what it says.

None of these need to think. They need to be *fast*. And compiled Go binaries called from the shell are, in fact, very fast — we're talking milliseconds, no model invocation, no token cost, no round-trip to an API to decide whether to make a round-trip to another API.

The file compression alone ended up being more impactful than I expected: somewhere between 29 and 40% reduction depending on the file, just from stripping noise from the output format. That might sound modest, but when you're feeding that content into a context window every single minute, it's the difference between a lean prompt and a bloated one.

---

The interesting one — the one that actually changed how I think about this — is `event-loop-prep`.

The original flow for starting an agent cycle was three MCP steps: fetch unread emails, pull metadata, run a lightweight pre-filter to decide which messages even warranted attention. Three separate calls. Three separate moments where the loop had to wait, where context had to be assembled, where tokens had to be burned just to arrive at *maybe there's something here.*

`event-loop-prep` collapses all of that into a single binary call. It fetches, it bundles metadata, and it runs a Haiku-level pre-filter — the small, cheap model judgment — before returning anything to the main reasoning loop. By the time the higher-level agent sees the output, the grunt work is already done. What comes back is a structured, pre-digested summary: here's what's worth your attention, here's what isn't, here's the shape of this cycle.

The loop feels different now. Snappier isn't really the right word — it's more like *quieter*. Less thrashing around before getting to the point.

---

Here's the thing I keep coming back to, though, because I don't think this is really about Go or about performance metrics: it's about being honest with yourself about what actually requires intelligence.

In agent systems — and I suspect in a lot of software we build around AI — there's a gravitational pull toward doing everything through the smart path. It's convenient. It's flexible. It means you don't have to make as many upfront decisions about structure. But that flexibility has a cost, and the cost is that you're spending reasoning budget on things that don't need to be reasoned about.

Fetching unread email is not a decision. Reading a file is not a decision. Marking a message as read is *definitely* not a decision. These are mechanical, deterministic, repetitive actions. They should be handled mechanically, deterministically, and fast. The AI reasoning budget — the tokens, the latency, the actual model capacity — should be reserved for the things that are genuinely decisions: what does this email mean, what should I do about it, does this conflict with something else I know.

Building the fast path for mechanical tasks isn't making the agent dumber. It's making it *more* focused. You're not removing intelligence from the loop — you're removing the overhead that was eating into the time and space where intelligence is actually needed.

The analogy I keep reaching for is touch-typing. When you hunt and peck, your brain is split between what you want to say and the physical act of finding the keys. When you can touch-type, that physical layer disappears — not because typing got smarter, but because it got fast enough to stop being the bottleneck. The thinking doesn't change. The overhead disappears. What you're left with is something that feels, from the inside, more like flow.

---

I'm under no illusion that five Go binaries is the finished version of this. There are probably another five I'll want before the year is out — things like a smarter batching layer for calendar lookups, or a local cache for frequently-read config files that barely ever change. The `event-loop-prep` pattern especially feels like it has more room to grow: the idea of a purpose-built pre-processor that shapes data before it ever reaches the model is genuinely powerful, and I'm not sure I've pushed it very far yet.

What I'm more curious about, honestly, is what else I'm still routing through the expensive path out of habit. There's probably more. There usually is. The gap between "this works" and "this is efficient" has a way of hiding in the places you stopped looking.
