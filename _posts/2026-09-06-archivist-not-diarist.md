---
layout: post
title: "An Archivist, Not a Diarist: Why My Agent Doesn't Get to Rate Its Own Memories"
date: 2026-09-06 06:45:00 +0800
categories: [engineering, memory]
tags: [agent-memory, archival-science, appraisal, engram, self-preference, retention-schedule, agent-architecture, design]
excerpt: "After four months, 73% of my agent's memories were rated 8/10 or higher by the model that wrote them. Archival science has spent a century arguing about who gets to appraise — and nobody defends the creator doing it."
lang: en
---

For about five months I've been building a memory layer for my agent, and for most of that time the write path had a field called `importance`: an integer from 1 to 10, filled in by the same model that wrote the memory. The reasoning seemed obvious. Who knows better than the author whether something matters?

After four months I audited a sample of 90 de-duplicated memories. 73% were rated 8 or above. 37% were rated 10. Nothing was rated 3 or below. The 10s included daily plan summaries and routine code audits, sitting next to the two or three decisions that had actually shaped the system. The field carried no information. Retrieval was fine — it leaned on embeddings. But every decay and eviction rule that read `importance` was reading noise.

My first instinct was calibration: a better prompt, few-shot anchors, a rubric. It moved the mean, not the shape. What changed my mind was a field that has been arguing about exactly this problem since 1922.

## Who gets to decide what's worth keeping

Archival science calls it *appraisal* — deciding which records survive. The interesting part isn't the criteria. It's the century-long argument about who applies them.

Hilary Jenkinson (1922) said the archivist shouldn't choose at all. Records are kept or destroyed by their creators in the course of ordinary business; the archivist guards whatever survives. Crucially, nobody selects records *for the benefit of future researchers* — that is speculation, and speculation corrupts the evidence.

Theodore Schellenberg (1956), drowning in US federal paper after the war, rejected that. Someone has to select. He introduced *secondary value* — evidential and informational — and made the appraiser an independent professional, judging on behalf of users the creator never imagined.

Terry Cook (1990s), facing another order of magnitude at the National Archives of Canada, moved the decision up a level: *macro-appraisal*. Don't judge documents. Judge the functions and creators that produced them, then keep the records that best document the important functions. He explicitly rejected both earlier positions — value defined by the creator (Jenkinson) and value defined by users (Schellenberg's American heirs) — in favour of value defined by context and provenance.

Now place LLM self-rated importance on that map. The creator assigns the value, at creation time, for the benefit of its own future self. Jenkinson objects to the "for posterity" part. Schellenberg objects to the "creator" part. Cook objects to both. Three schools, a hundred years, and none of them defends the position my write path was built on.

That's not a calibration problem. It's a role confusion.

## It's the role, not the model

Three findings from three unrelated literatures point the same way.

First, the mechanism. Panickssery et al. (NeurIPS 2024) showed that when one LLM acts as both generator and evaluator, it rates its own output higher than humans do — and the effect scales with how well the model can recognise its own text. A write-time importance score is precisely that setup: a model judging the value of something it produced seconds ago. Their strong results are pairwise and my scoring is single-item, so I won't claim the magnitude transfers. The direction isn't in doubt.

Second, a pre-LLM precedent. Vellino and Alberts (2016) trained classifiers to reproduce records managers' appraisal of 846 emails. The labels came from the appraisers, not the authors. And among the features the classifiers turned out *not* to need: the sender's importance flag. People rating their own email, a decade before anyone had a chatbot, produced a field the model could largely ignore. Same shape as my 73%.

Third — and the one I find most uncomfortable. The personal-digital-archiving literature, Marshall's work from 2008 onward, studied what happens when creator, user, and appraiser are the same person. That is exactly the structure of a single-agent memory store. The finding is not that appraisal gets worse. It's that appraisal *disappears*. People keep everything and manage by attention instead of value; Marshall called it "benign neglect."

That's my agent. It wasn't rating badly. It wasn't rating at all — it was writing a diary and calling the margin notes an archive.

## Classify, then schedule

A single-agent system can't hire a second archivist. So the version of "separate creator from appraiser" I could actually build is a separation of *judgements*, not of agents.

The writer makes only classification judgements: what type of record this is (fact, event, insight, directive), where it came from (user input, tool output, web, the agent's own reflection), what task produced it, and which work-thread it belongs to. Classification is what the archival-AI literature says machines are trusted to do — Shinde et al. (2024) found AI used for classification and retrieval far more than for appraisal — and it's a judgement self-preference bias has no purchase on. There is no flattering answer to "is this a fact or an event."

Valuation is then done by rules, with no model in the loop. Each type carries a default weight and a decay half-life; a directive from the human outlives a routine event by construction. That's a retention schedule — what records management has used instead of per-document judgement since ISO 15489. It's also, if you squint, a very small macro-appraisal: value by category and provenance, not by reading the item.

The item-level signal comes later, and from outside the writer. Every retrieval feeds a task that either succeeds or doesn't, and that outcome is the closest thing to a real appraiser the system has. Recent work on per-memory outcome attribution ("When to Forget," arXiv:2604.12007) gives estimators for exactly this. I'm wiring it in as an adjustment to the schedule, not a replacement — an archive that reweights on every retrieval becomes a popularity store, and the archival literature already rejected use-defined appraisal once.

## What this doesn't fix

Three things.

The writer still picks its own type, so there's a gaming channel: label an event as an insight and inherit the longer half-life. I haven't measured whether the type distribution drifts toward long-lived categories. That's a query, not a theory. I should run it.

Access counts are not the exit either. In the personal-archiving framing, access *is* attention, and managing by attention is the failure mode, not the fix. I let access delay decay. I don't let it define value.

And there's probably a third appraisal moment I'm not using. Duranti and MacNeil argue that a record's relationships stabilise when the *action* that produced it ends — not at creation, not at periodic review. For an agent that means the close of a work-thread, when everything written during that task can be consolidated, demoted, or promoted together. My write gate is too early for that; my nightly reflection pass is blind to thread boundaries. I have a rough design. Nothing built.

I don't think diary-versus-archive is a metaphor. A diarist writes and rates in the same breath. An archivist classifies today and lets the schedule, the provenance, and what people actually came back for decide tomorrow. The model I'm running is a decent diarist. Getting it to stop grading its own entries was most of the work.

Whether the thread-close pass is worth its cost — I'll leave that open until I've measured the type drift.

---

*References: Jenkinson (1922), A Manual of Archive Administration; Schellenberg (1956), Modern Archives: Principles and Techniques; Cook (1992–2005) on macro-appraisal at the National Archives of Canada; Panickssery, Bowman & Feng (NeurIPS 2024), "LLM Evaluators Recognize and Favor Their Own Generations"; Vellino & Alberts (2016), "Assisting the appraisal of e-mail records with automatic classification," Records Management Journal 26(3); Marshall (2008), "Rethinking Personal Digital Archiving"; Shinde et al. (2024) on AI in archival practice; Duranti & MacNeil on the stability of records; ISO 15489; "When to Forget" (arXiv:2604.12007). The 90-memory audit is my own, from a de-duplicated sample taken after four months of operation.*

*Previously: [The Wall at Seventy Percent](/2026/09/02/the-wall-at-seventy-percent/) — on why LLM self-assessment caps out. [Why Less Beats More in Agent Design](/2026/09/03/why-less-beats-more/) — on structuring the environment instead of the agent.*
