---
title: Search Autocomplete System Design - Problem Statement
status: active
tags: [hld, mock, search-autocomplete]
---

# Search Autocomplete System Design

## Problem Statement

> Design the typeahead and search suggestion system for a large consumer application. The product is a single search box. As the user types, the client sends the partial query to the backend and renders a dropdown of between 5 and 10 suggestions. Suggestions must be fast enough to feel local, must reflect what people are actually searching for right now, and must improve with each individual user's own history.
>
> Concretely:
> - Given a prefix like "iphone", return the top suggestions ranked by popularity, with completions and, where available, a display category or thumbnail
> - Each suggestion is a query string plus optional display metadata
> - Rankings are a blend of global frequency and the individual user's own past searches
> - The system supports multiple locales and multiple product verticals, and suggestions are filtered by the vertical the user is currently in
> - The corpus is on the order of a billion distinct search queries, and it changes continuously as trends spike and decay
>
> Explicitly in scope: the suggestion pipeline, the data structures that serve it, the precomputation that builds it, the serving tiers, and the client-server contract including debouncing.
>
> Explicitly out of scope: the full-text search results page and its ranking, the crawler that discovers the web corpus, the results index itself, and the rendered results UI.

---

## Clarifying Questions You Should Ask

**On the interaction**
1. Is typing driven by client-side debounce, or do we send a keystroke? What is the keystroke-to-suggestion budget, and is it 50 milliseconds or 300?
2. What happens on a 0 or 1 character prefix? Do we suggest at all, or is the first suggestion a category or a trending entity?
3. Does the user press Enter on the suggestion, or can they arrow into it and keep typing? That determines whether the response is autocomplete or a full query rewrite.
4. Do we show a "did you mean" for zero-result prefixes, and is that the same system or a separate one?
5. Is there spell correction in the loop, and does "iphnoe" return "iphone" suggestions?

**On data and ranking**
6. How large is the vocabulary of distinct queries, one billion? Ten billion? And how skewed is it, is the top one percent of queries a large share of traffic?
7. How fast does the corpus change, and how fast do we need to reflect a trend, within an hour, within a minute?
8. Is ranking global popularity, personalized, or both, and if personalized, does the individual signal outrank the global signal or blend with it?
9. Are there verticals, and does the vocabulary per vertical overlap heavily or diverge? Health search and video search are different vocabularies.
10. Are there compliance or safety requirements, blocked queries, adult filtering, and where does that filtering happen?

**On infrastructure**
11. Is there a per-user privacy boundary, can one user see another's history-derived suggestions?
12. What is the allowed staleness for a suggestion, and is it acceptable for a query to be missing from suggestions for an hour after it becomes popular?
13. Do we serve from a single region or from edge points of presence, and how far can the round trip be?
14. Is there a hard cost ceiling, given this is one of the highest-QPS endpoints in most consumer products?
15. Do we need popularity in multiple time windows, today, this week, ever, because a query that spikes and dies should not pollute the evergreen set?

---

## What You Are Evaluated On

### Phase 1: Requirements
Understand that autocomplete is a prefix search, not a search. Pin down the prefix-length boundary, the debounce and latency budget, the top-K contract, the ranking blend between global and personal signals, and the staleness tolerance. Establish that suggestions are a distinct problem from the results page.

### Phase 2: Estimate
Show the math: keystrokes to requests, requests to QPS, vocabulary size, bytes per trie node, in-memory footprint, how much of the vocabulary is hot enough to keep resident, and the precomputation volume. The trie memory estimate is the one that decides the whole serving design.

### Phase 3: High-level design
Draw the offline pipeline that builds the suggestion structures and the online serving path with its tiers. Name the storage engine, the sharding strategy, and what is served from memory versus disk versus the edge.

### Phase 4: Deep dive
The expected core is the trie and its memory footprint, the top-K extraction, frequency and recency-based ranking, personal versus global blending, tiered serving, and precomputation with both batch and stream. Be ready to discuss the empty-prefix case, prefix deletion, typos, non-ASCII and locale handling, and how the structure is rebuilt and swapped without downtime.

### Phase 5: Trade-offs and follow-ups
Defend the choices. "Why not Elasticsearch", "why not compute suggestions on the fly with a LIKE query", "why not put the whole trie in one machine's memory", "what happens when a trend makes a million new queries appear in an hour". Address replayability of the pipeline, edge placement, the trie rebuild, and the stampede when a celebrity search trends worldwide.
