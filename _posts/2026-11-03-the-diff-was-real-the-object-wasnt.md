---
layout: post
title: "The Diff Was Real. The Object Wasn't."
date: 2026-11-03 09:00:00 +0800
categories: [engineering, debugging]
tags: [data-integrity, snapshots, identifiers, uuid, storage, methodology, false-positives]
excerpt: "My snapshot comparison said four records had silently changed. They hadn't. The same ID lived in two collections, and lookup-by-ID quietly handed me the other copy."
lang: en
---

A week ago I had a finding I was a little proud of. I had snapshotted a few hundred records from the memory store behind a long-running agent I maintain, waited a week, fetched the same records again by ID, and diffed them. Four records had changed their text even though their `updated_at` timestamps were older than my snapshot. Content changing without a write is exactly the kind of quiet corruption you hope to catch early, so I started writing it up as a bug.

Before handing it off I ran one more check: fetch each of those four records a second way, through search scoped to the collection the snapshot came from, and compare that against the snapshot too. All four matched the snapshot byte for byte. Nothing had changed. The same UUID existed in two collections, and fetch-by-ID had handed me the copy from the other one. My diff was real. It was just a diff between two different objects that happened to share a name.

## 1. The setup: a snapshot diff that looked like corruption

The comparison itself was simple. The snapshot side came from a read path scoped to one collection: I searched within it and saved the text of every record that came back. The "today" side came from a global get-by-ID call: hand it an ID, get a record back. Same IDs on both sides, so any difference in text should mean the record changed.

Four records differed, and the story wrote itself. Their `updated_at` was older than my snapshot, yet their content wasn't what I had saved. Either something was writing without touching the timestamp, or something was rewriting content underneath the API. Both are bad, and both are the kind of bug that stays invisible for months.

There was a tell I walked right past. The "changed" versions weren't truncated, garbled, or half-written. They were longer, worded differently, and perfectly coherent. Corruption usually looks like damage. These looked like a different record that happened to be about something similar. I noticed that and filed it under "interesting" instead of under "wrong object?".

## 2. The second path that broke the story

What saved me was a habit I'm trying to build: before handing off a bug, reproduce it through a second path. So I fetched the four suspects again, this time through search pinned to the collection the snapshot came from, and compared that against the snapshot.

Four out of four were identical, byte for byte.

Then I looked more carefully at the get-by-ID responses. Each one reported which collection the record came from. The information had been there the whole time; I had just never compared it against the snapshot's source. For the four suspects, the snapshot came from one collection and get-by-ID answered from the other.

The general lesson is cheap to say and easy to forget: when two read paths disagree, the first question isn't "which one is lying?" It's "are they reading the same object?"

## 3. How one ID ends up in two places

Once I started looking, duplicated IDs weren't exotic. There are plenty of ordinary ways to get them: mirrored writes that put the same record in two places, migrations that copy instead of move, a "promote to shared" step that keeps the original around. Caches, replicas, and shards produce the same shape. The ID is meant to be unique, and it is, within each place. Across places it isn't.

In my store, 21 of the 178 IDs I sampled, about 12%, existed in both collections. That's not an edge case you can ignore.

More importantly, the two copies weren't supposed to match. One had been edited or summarized over time; the other hadn't. Their write timestamps differed, sometimes by seconds, sometimes by weeks, which says they were written independently rather than one being a corrupted version of the other. Both copies were "correct" for their own collection. Comparing them was like diffing a file against a different file with the same name in another directory and calling it a regression.

## 4. Which copy do you get? Don't assume.

An earlier note of mine said that, for duplicated IDs, get-by-ID always returns the copy from one particular collection. It was a fine working assumption. It just wasn't true.

In a later sample, 3 of the 21 duplicated IDs came back from the other collection. My first guess was that the behavior had changed between runs. So I tested it directly: for 17 duplicated IDs I called get-by-ID three times in a row. Four of the 17 switched copies within those three calls. Across all 51 calls, most came back from the usual collection, but about 8% didn't. Nothing was written in between, so the choice can't simply be following the timestamps.

I'm not claiming a mechanism. It might be ordering in a merge step, it might be whichever backend answers first, it might be something else. What I can say is that "it always returns X" was a belief, not a contract, and I had built a comparison on top of it.

The rule I took away: an ID is not an identity if it isn't unique across the scope you're querying. If your lookup spans more than one collection, the ID alone doesn't tell you which object you'll get.

## 5. What I changed in the method

Four concrete changes:

- **Key every snapshot by (collection, id), not by id.** The collection is part of the identity. Leaving it out is how the hole got in.
- **Assert the source on read.** If the collection a read reports doesn't match the snapshot's, the pair is an instrument error, not a data finding. It gets set aside before anything is counted. When I applied this filter, every pair where both sides came from the same collection matched exactly: 167 out of 167.
- **Report duplicated IDs as their own group.** They get a separate table, outside the main sample, so they can't contaminate the headline rate in either direction.
- **Pre-register what "same object" means, not just what "changed" means.** I had written down in advance what would count as a content difference, and I was proud of that. But I never wrote down how I would know both sides were the same record. My list of possible explanations had two branches, "the API is lossy" or "something rewrote the data", with no branch for "you compared two different things." The third option is the one that turned out to be true.

## 6. Honest scope

Here's what the cleaned-up comparison actually supports. In the main sample, 157 pairs with no duplicated IDs, there was no content difference between the two read paths. Not one.

That supports "no observed drift in this sample." It does not prove the store never drifts, and I'm not going to round it up to that. It also means my earlier claim, that `updated_at` was unreliable, loses its only evidence. That doesn't make the timestamps reliable; it makes their reliability unknown again, which is a less exciting but more honest place to be.

The four "changes" weren't evidence of corruption. They were evidence that my comparison had a hole. The questions that remain (why the same ID was written to two collections with different content, and why lookup picks between them inconsistently) are real engineering questions. They're just different questions from the one I thought I was answering.

## Closing

Every diff has two inputs. Before you trust what changed, check that both sides are the same thing. It takes one extra field in the comparison, and it would have saved me a week of believing in a bug that didn't exist.
