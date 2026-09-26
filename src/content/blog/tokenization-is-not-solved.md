---
title: 'Tokenization is not a solved problem'
description: 'For English it is invisible infrastructure. For an agglutinative language it is the thing standing between you and a working model — and it shapes cost, context and quality all at once.'
pubDate: 2026-09-12
tags: ['Tokenization', 'Low-resource NLP', 'Azerbaijani', 'NLP']
---

If you work in English, the tokenizer is something you never think about. It came with the model, it splits text into pieces that mostly look like words, and it has never been the reason your experiment failed.

Start working in an agglutinative language and it becomes the first thing you have to fix.

## What agglutination does to a subword vocabulary

Azerbaijani, like Turkish and a long list of other languages, builds meaning by stacking suffixes onto a root. Plurality, case, possession, tense, negation, question — each is a suffix, and they compose. One root can generate an enormous number of surface forms, all perfectly ordinary words a speaker uses without thinking.

A subword vocabulary learned mostly from English text has no reason to have learned any of those suffixes as units. So it falls back on what it does have: short, frequent character sequences that happen to appear. A single common word gets shattered into four or five fragments, none of which correspond to a morpheme, and the boundaries land in different places for different inflections of the same root.

The model can still learn. It is just being made to reconstruct the morphology of the language from fragments, using capacity that could have gone somewhere more useful.

## Fertility, and why it is not just an aesthetic complaint

The number to watch is **fertility**: average tokens per word. English with an English-trained tokenizer sits near 1.3. A poorly-matched tokenizer on a morphologically rich language can be two or three times that.

That ratio is not cosmetic. It multiplies through everything:

- **Context windows shrink.** A fixed token budget holds proportionally less actual text. Your retrieval chunks fit less content, your prompts fit fewer documents.
- **Costs scale directly.** Inference priced per token means you pay the fertility multiplier on every request, forever.
- **Sequence lengths grow**, and attention cost grows with them.
- **Training sees less.** The same compute budget covers proportionally less text.

A tokenizer that is twice as fertile on your language is a permanent, compounding tax on everything downstream. It is one of the few problems where the fix is cheap and the payoff is structural.

## Building one instead

Training a tokenizer on in-language text is not hard and the effect is immediate. The decisions that matter:

**Algorithm.** BPE, WordPiece and Unigram make different trade-offs, and which is best is genuinely empirical for a given language. Unigram's probabilistic segmentation tends to handle morphology more gracefully than greedy merging, but this is worth testing rather than assuming.

**Vocabulary size.** The core trade-off. Larger vocabulary means lower fertility but more embedding parameters, most of them rarely used. There is a knee in the curve where fertility stops improving much; find it on your own corpus rather than copying a number from a paper about a different language.

**Normalisation.** Decide explicitly how you handle case, diacritics and the several ways the same character can be encoded. Azerbaijani has letters — dotted and dotless i in particular — where careless case handling silently corrupts text. Get this wrong and you have introduced a bug that will be nearly invisible in your metrics and very visible to a native speaker.

**Corpus composition.** The tokenizer learns the distribution you show it. If your corpus is 90% news and you will deploy on legal text, the vocabulary will be optimised for the wrong register. For a domain-specific system, weight the corpus toward the domain.

## The domain layer on top

This is the part that bit me specifically. Legal text has its own vocabulary — terms of art, statutory references, formulaic constructions — that a general-corpus tokenizer has never seen often enough to keep whole.

Those are exactly the tokens that carry the meaning in a legal query. Having them fragmented into character soup is not a small inefficiency; it is the model losing its grip on the terms that decide whether a retrieval is right or wrong.

## What this generalises to

Tokenization is the clearest case of something that happens all over the stack when you leave the well-resourced centre: a component that is invisible infrastructure for English becomes a design decision you have to make yourself.

You learn what it was doing for you by losing it. That is genuinely a good way to learn, even if it is a slow one.

If you are starting an NLP project in a language that is not English, measure your tokenizer's fertility on your own text before you do anything else. It takes an afternoon, and it will tell you whether you are about to pay a 2× tax on every experiment you run for the next year.
