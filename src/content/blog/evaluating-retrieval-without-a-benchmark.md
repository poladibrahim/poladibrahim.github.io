---
title: 'Evaluating retrieval when no benchmark exists'
description: 'If your language or domain has no public benchmark, you have to build the ruler before you can measure anything. Notes on building relevance judgements that are worth trusting.'
pubDate: 2026-09-24
tags: ['Evaluation', 'Retrieval', 'Low-resource NLP', 'Research']
---

Here is the uncomfortable position you end up in when you build search for a domain nobody has benchmarked: you can ship a change, watch it feel better on the five queries you keep typing, and have no idea whether you improved the system or broke it for everyone else.

I spent a long time in that position. This is what got me out of it.

## Why "it feels better" fails

The five queries you type are not a random sample. They are the queries you already know the system handles, plus the one that embarrassed you in a demo last week. You have been optimising against them for months, which means they are the queries most likely to already work.

Worse, the failure mode of retrieval is invisible from the inside. If the right document is not in the results, nothing looks wrong — you just see plausible documents that are not the answer. You need the ground truth to notice, and ground truth is exactly what you do not have.

## What a relevance judgement actually is

The unit you need is boring: a `(query, document, label)` triple. Someone who understands the domain looked at this document, in the context of this query, and said "yes, this answers it" or "no, it doesn't."

Two things make this hard in practice.

**You cannot judge every document.** With a corpus of any size, exhaustive labelling is impossible. The standard trick is *pooling*: run several different retrieval systems over each query, take the union of their top-k results, and judge only that pool. Anything no system ever surfaced is assumed non-relevant. This is an approximation, and it has a known bias — a future system that finds something genuinely relevant that none of your original systems retrieved will be unfairly penalised. You accept this and stay aware of it.

**Binary is often not enough.** "Relevant" hides a real distinction between *this is the article the question is about* and *this is related background that a reader might want*. Graded judgements (say 0–3) cost more to produce but let you use metrics that care about ordering rather than just membership.

## Getting queries that are not yours

Your evaluation set is worthless if the queries came out of your own head, because your head is where the system's assumptions came from.

Sources that worked better:

- **Real user queries from logs**, if you have them. Sample across the frequency distribution, not just the head — the tail is where systems fail and the tail is most of the traffic.
- **Domain experts asked to write questions**, given a document and asked what someone would ask to find it. This inverts the task and produces phrasings you would never invent.
- **Questions that exist in the wild** — forums, FAQs, whatever your domain's equivalent is of people asking each other things.

The queries should be phrased the way users phrase them, including when that is vague, misspelled, or uses the wrong term. A benchmark made only of well-formed questions measures a system nobody is using.

## Picking a metric that matches the job

Metric choice is not a formality; different metrics reward different behaviour.

- **Recall@k** at the point where candidates enter your reranker. This is the ceiling on everything downstream — if the right document is not in the candidate set, no reranker can save you.
- **nDCG@10** for the final ranking. It cares about graded relevance and about position, which is what you want when users read from the top.
- **MRR** if there is exactly one right answer and finding it fast is the whole task.

I measure recall at the merge boundary and nDCG at the end, because those are two genuinely different questions: *did we find it* and *did we put it first*. A change can improve one and hurt the other, and a single end-to-end number will hide that from you.

## Guard against the set you built

A fixed evaluation set becomes a target, and targets get overfitted. Some defences:

**Hold some of it back.** Split into a development set you tune against and a test set you touch rarely. Yes, this is basic. Yes, it gets skipped constantly on internal projects because it feels like ceremony when there are only 300 queries.

**Re-judge periodically.** Labels drift, especially in a domain where the underlying documents change. A judgement made against last year's version of a document may be wrong now.

**Track agreement between judges.** If two domain experts disagree on 30% of pairs, your metric has a noise floor and differences below it mean nothing. Knowing that number stops you shipping changes that are indistinguishable from noise.

## The part nobody tells you

Building this is slow, unglamorous, and looks like you are not working on the model. It takes weeks. You will be tempted, repeatedly, to skip it and get back to the interesting part.

Every improvement claim you make afterwards rests on it. Without it you are not doing engineering, you are doing taste. Taste is worth something, but it cannot tell you whether the reranker you spent a month on is earning its latency.

If I started a new retrieval project in an unbenchmarked domain tomorrow, I would build the evaluation set before writing a line of model code. Not because it is virtuous — because everything after it is guesswork otherwise.
