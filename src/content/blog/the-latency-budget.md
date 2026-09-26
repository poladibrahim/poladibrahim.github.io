---
title: 'The latency budget: what actually makes a RAG pipeline fast'
description: 'A smaller model is usually the last thing that helps. Where the time really goes in a retrieval pipeline, and the three unglamorous changes that move it.'
pubDate: 2026-09-18
tags: ['Production', 'RAG', 'Performance', 'MLOps']
---

The first instinct when a retrieval pipeline is too slow is to reach for a smaller model. In my experience that is close to the last thing that helps, and it costs you quality on the way.

## Measure the stages separately

A RAG pipeline is not one thing. A request passes through, roughly:

1. Embed the query
2. Dense search
3. Lexical search
4. Merge candidates
5. Rerank
6. Generate

An end-to-end p95 tells you the total is too high. It tells you nothing about which stage to fix, and intuition about this is unreliable — I have been wrong about which stage dominated more than once.

Instrument each stage separately, and look at **p95 and p99, not the mean**. The mean is dominated by the fast requests you were never worried about. The slow tail is what users experience as "this is broken."

## Where the time usually goes

Once you can see the breakdown, the pattern is often the same: **reranking dominates**, and it dominates in proportion to how many candidates you feed it.

The cross-encoder is the expensive stage by design — it runs a forward pass per `(query, candidate)` pair, so its cost is linear in candidate count, while the retrievers are essentially index lookups. Doubling your candidate set doubles your rerank time and does not double your quality.

Which makes **candidate set size the single most consequential number in the pipeline**. It is a real hyperparameter, it trades quality against latency directly, and it deserves a proper sweep rather than whatever value you typed in during the first week. Sweep it, plot recall against latency, and pick the point where recall stops climbing. It is usually lower than you expect.

## The three changes that actually move it

### Batching

Running embedding or rerank calls one at a time leaves most of your GPU idle. Batch them — both *within* a request, where you have many candidates to score at once, and *across* concurrent requests, by collecting requests over a small window and running them together.

The cross-request version costs you a few milliseconds of added wait per request and can buy back far more in throughput. It is the highest-leverage change in most pipelines, and the one most likely to be missing.

### Caching

Query traffic is not uniformly distributed. In any real system a meaningful fraction of queries are repeats or near-repeats of something asked recently.

Worth caching, roughly in order of how much they pay off:

- **Document embeddings** — these should never be recomputed at query time.
- **Query embeddings**, keyed on the normalised query string.
- **Rerank scores** for a `(query, document)` pair.
- **Whole result sets**, where staleness is acceptable.

The last one needs care. A cached result set is wrong the moment the underlying corpus changes, so it needs invalidation tied to your indexing pipeline, not just a TTL you hope is short enough.

### Parallelism

The dense and lexical retrieval passes have no dependency on each other. If you are running them sequentially, you are paying for both when you could pay for the slower one.

This is free latency and it gets missed constantly, usually because the code was written one stage at a time and nobody went back to look at the shape of it.

## Things that helped less than expected

**A smaller reranker.** Cheaper per pair, worse at the job it exists to do. If you are going to trade quality for speed, cutting the candidate set is a better trade — it keeps the model's judgement intact and just asks it fewer questions.

**Quantisation, sometimes.** Real speedups, but the quality effect is task-specific and you will not know which you got without your evaluation set. Do not assume it is free.

**Rewriting things in a faster language.** In a pipeline dominated by model forward passes and network calls, the Python overhead is rarely the problem. Profile before you believe otherwise.

## Set a budget and hold to it

Decide what the pipeline is allowed to cost — say 500 ms at p95 — and treat it as a constraint rather than an aspiration. Then every proposed addition has to justify its share.

This is the thing that stops the slow creep where each change adds 40 ms and nobody objects, because no single change is the problem. Six months later the pipeline takes two seconds and you cannot point at what did it.

The unglamorous version of performance work is: measure per stage, tune the candidate set, batch, cache, parallelise, and keep a number you refuse to exceed. None of it is research. All of it is the difference between a demo and something people will actually use.
