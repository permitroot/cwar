# CWAR: Compiled Wiki-Augmented Retrieval

A compile-first RAG architecture: knowledge is compiled before it is retrieved, and the byproducts of compilation drive retrieval scoring.

> Author: Geunho Kim  
> Date: April 12, 2026  
> Repo: [cwar](https://github.com/permitroot/cwar)  
> Builds on: [LLM Wiki](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) by Andrej Karpathy

---

## Why this exists

When Karpathy published LLM Wiki, a nagging question surfaced immediately: what happens at scale?

He's upfront about it. The `index.md` navigation model works well "at moderate scale (~100 sources, ~hundreds of pages)." When you outgrow the index file, he recommends qmd — a local search engine for markdown with hybrid BM25/vector search and LLM re-ranking. That's a good answer to the navigation problem. But it doesn't address what's actually breaking as the wiki grows.

The real ceiling isn't navigation. It's that the entire wiki (or index plus selected pages) must fit in the LLM's context window. At roughly 50,000–100,000 tokens — maybe 100–200 wiki pages — you hit it. You either truncate (losing coverage) or summarize (losing the accumulated synthesis that makes the wiki worth having in the first place). The compounding stops.

The obvious fix is embedding-based RAG. Dozens of community implementations — SamurAIGPT's llm-wiki-agent, agent-wiki, SwarmVault — have done exactly this. They work. But almost all of them make the same decision: they embed compiled wiki pages the same way they'd embed raw document chunks, as text, and discard everything else compilation produced.

That's the gap CWAR addresses.

When an LLM compiles a source into a wiki page, it generates more than text. It produces cross-references to related concepts, a count of corroborating sources, explicit questions the page can answer, and a record of source provenance. These are signals about the *quality* and *context* of the content — exactly what you want shaping retrieval. When they get thrown away at the index boundary, you're building a vector database that treats a well-sourced, multiply-corroborated, lint-verified page identically to a single-source draft that was never reviewed.

CWAR's principle: **compilation metadata belongs in the retrieval scoring function, not discarded at the embedding step.** A better-maintained wiki produces better search results. The retrieval index improves over time rather than just growing.

---

## The core idea

A raw chunk is a decontextualized fragment. "Revenue grew 3% quarter-over-quarter" — from which company? Which quarter? What was the prior trend? Contextual Retrieval (prepending LLM-generated context before embedding) helps, but you're still retrieving fragments, and the LLM still has to reconstruct synthesis at query time, on every query.

A compiled wiki page is a different kind of object — the canonical knowledge artifact about a topic. It already contains cross-document synthesis, explicit cross-references, flagged contradictions, a TL;DR optimized for top-level matching, and a list of questions it can directly answer. When these pages enter a vector database with their compilation metadata intact and flowing into retrieval weights, the system operates over knowledge artifacts, not text fragments.

This is the substance of CWAR: **the LLM Wiki compilation step as the construction layer for a vector retrieval system**, where the two reinforce each other instead of operating independently.

---

## Position: LLM Wiki, GraphRAG, and CWAR

CWAR sits between LLM Wiki and GraphRAG. Understanding the trade-offs clarifies what it is and isn't.

**LLM Wiki** is radically simple — markdown, Git, LLM, no infrastructure. It works well for personal projects under ~100 pages, but can't scale past context window limits and can't do cross-cutting search.

**GraphRAG** (Microsoft) brings entity-relationship extraction, Leiden community detection, hierarchical reports, and Local/Global Search. It's academically validated (average 35% accuracy improvement over vanilla RAG in benchmarks). But indexing costs 100–1,000x more, it requires a graph DB alongside the vector DB, and relationship triples struggle with complex causal chains. LazyGraphRAG cuts indexing costs 99.9% by deferring processing to query time, at the cost of query latency.

**CWAR** reuses the compilation work already done by LLM Wiki. The key cost insight:

| Stage | GraphRAG | CWAR |
|-------|----------|------|
| Document understanding | LLM call (entity extraction) | LLM call (wiki compilation — already happening) |
| Structuring | LLM call (relation summaries, community reports) | Unnecessary (produced during compilation) |
| Embedding | Chunks + entities + reports | TL;DR + sections + questions_answerable |
| Extra infrastructure | Graph DB + vector DB | Vector DB only (pgvector) |

GraphRAG runs expensive processing *for retrieval*. CWAR leverages *byproducts of compilation that already happened*. For individual users and small teams, this cost difference is decisive.

---

## Architecture

Five components. The first two are Karpathy's LLM Wiki, essentially unchanged.

**Raw sources.** Immutable. PDFs, web clips, transcripts, API exports, code. The LLM reads from here; it never writes here.

**The wiki.** LLM-authored markdown in `wiki/`, with `index.md` as the catalog and `log.md` as the chronological record. The one addition CWAR requires: **every wiki page carries YAML frontmatter from the moment of creation.**

**The adaptive compiler.** Not all sources need the same compilation strategy. Source type detection drives compilation depth:

- **API documentation**: structure-preserving — TL;DR, code examples, parameter tables
- **Incident reports**: causal chain focus — timeline, cause → impact → response → prevention
- **Research papers**: claim-evidence structure — claims, evidence, limitations
- **Meeting notes / chat logs**: decision extraction — decisions, action items, context

Each type uses a different compilation prompt and frontmatter emphasis. The compiler also calculates `structural_confidence` — a reproducible score derived from objective signals rather than LLM self-assessment (detailed in the scoring section).

**The embedding pipeline.** After each compilation pass, changed pages are indexed three ways: TL;DR embedding for fast top-level retrieval, section-level embeddings for detailed content matching, and per-question embeddings from `questions_answerable` for direct question-to-document retrieval. Frontmatter becomes filterable scalar metadata. Cross-references become adjacency edges for lightweight graph traversal.

**Hybrid retrieval with lightweight graph.** Vector similarity, BM25, and metadata filtering run in parallel. Vector search finds entry points; 1–2 hop graph traversal over cross-reference edges expands the neighborhood. Results are fused via Reciprocal Rank Fusion, passed through a reranker, and scored with the structural confidence function.

The lightweight graph layer deserves emphasis. GraphRAG requires a full entity extraction pipeline and a dedicated graph database. CWAR's cross-references — already generated during compilation — *are* the graph. Storing them as adjacency lists in pgvector (no Neo4j needed) gives Local Search–equivalent capability at zero additional LLM cost.

---

## Wiki page format

The YAML frontmatter enables everything downstream:

```markdown
---
title: "Stripe Payment Intents API"
category: "payment-infrastructure"
entities: ["Stripe", "PCI DSS", "PaymentMethod"]
structural_confidence: 0.82
cross_references: ["stripe-connect-express", "pci-compliance", "flutter-payment-flow"]
source_count: 5
last_compiled: "2026-04-12T09:30:00Z"
questions_answerable:
  - "What is the two-step flow for Payment Intents?"
  - "How does client-side tokenization affect PCI DSS scope?"
  - "Which Stripe objects survive a failed payment attempt?"
compilation_version: 3
has_contradictions: false
source_type: "api-docs"
---

# Stripe Payment Intents API

## TL;DR
Payment Intents is Stripe's server-side payment creation API, following a two-phase
flow: the client tokenizes the PaymentMethod, the server confirms the intent.

## Body
[synthesized knowledge, drawn from all ingested sources]

## Cross-References
- [[stripe-connect-express]]: settlement flow for Express accounts
- [[pci-compliance]]: why tokenization limits PCI DSS compliance scope

## Data Gaps
- Multi-currency settlement for Connect Express accounts not yet confirmed
- Radar fraud detection integration needs additional sources

## Sources
- raw/stripe-docs-payment-intents.md (ingested: 2026-03-15)
- raw/pci-dss-v4-summary.md (ingested: 2026-02-20)
```

Note `structural_confidence` instead of `confidence`. The distinction matters — see the next section.

`questions_answerable` comes from Azure's Chunk Enrichment research: pre-generating "questions this page can answer" and embedding them separately consistently outperforms embedding answer text alone for question-answering retrieval. Applied to compiled pages, it's particularly effective because the LLM generates questions about content it wrote and deeply understands.

This schema also works with Obsidian Dataview: live queries over `structural_confidence`, `source_count`, and `category` give a real-time view of knowledge base health.

---

## How retrieval scoring works

CWAR's scoring function:

```
final_score = vector_similarity
            × confidence_weight(structural_confidence)
            × freshness_weight(staleness_days)
            × authority_weight(cross_ref_inbound, source_count)
            × popularity_weight(retrieval_hit_count, page_age_days)
```

### Structural confidence

The original LLM Wiki uses LLM-generated `confidence` scores — a natural choice for a system where the LLM is the author and reader. But in CWAR, confidence directly drives retrieval scoring, and LLM self-assessment is inherently uncalibrated: the same content compiled twice might yield 0.85 or 0.92. When this variance flows into ranking, it introduces noise at the system level.

CWAR replaces subjective confidence with a reproducible structural score:

```
structural_confidence = weighted_sum(
    source_count_score,         # 1 source → 0.3, 3 → 0.7, 5+ → 0.9
    cross_ref_inbound_score,    # inbound citations from other pages
    lint_pass_recency,          # time since last successful lint
    contradiction_penalty,      # -0.2 if has_contradictions
    human_review_bonus          # +0.1 if manually reviewed
)
```

All inputs are objective and deterministic. Compile the same page twice — same structural confidence.

### Freshness weight

Decays gradually as `staleness_days` grows. Pages don't vanish from results when they age, but recently recompiled pages get a nudge. The decay rate should be domain-dependent: fast-moving API docs decay faster than theoretical foundations.

### Authority weight

The PageRank equivalent. Pages with many inbound cross-references carry more authority than peripheral pages, regardless of semantic similarity. `source_count` contributes: a claim supported by five sources outranks one supported by one.

### Popularity weight with bias correction

Every retrieval increments `retrieval_hit_count`, reinforcing pages users actually find useful. But uncorrected popularity creates filter bubbles — frequently retrieved pages get more exposure, rarely retrieved pages get buried regardless of quality.

Two corrections: **temporal decay** (recent retrievals weighted more than old ones) and **cold-start boost** (new pages receive a baseline score for a configurable period so they have a chance to prove useful before popularity ranking takes effect).

### Default weight presets

The relative importance of these factors varies by domain. CWAR provides presets as starting points:

| Profile | Confidence | Freshness | Authority | Popularity |
|---------|-----------|-----------|-----------|------------|
| Technical docs | 0.25 | 0.30 | 0.30 | 0.15 |
| Research notes | 0.30 | 0.10 | 0.40 | 0.20 |
| Operational runbooks | 0.20 | 0.40 | 0.20 | 0.20 |

These are tunable. The point is providing a reasonable starting configuration rather than requiring every deployment to discover weights from scratch.

---

## The four feedback loops

CWAR's most consequential property: **the system improves without dedicated maintenance work** beyond what you're already doing.

**Ingest-compile.** A new source doesn't just create a new page. The LLM re-reads related pages and revises them, updating cross-references, incrementing `source_count` on corroborated pages, detecting contradictions. All of this propagates to the vector index on the next embed pass. A single new source might touch fifteen pages.

**Query-feedback.** Every retrieval event updates hit counts. More powerfully: comparison tables, cross-reference analyses, and connections discovered during query answering go back into the wiki as `query-result` pages with full frontmatter. Your explorations compound into the knowledge base, not into chat history that disappears.

**Lint-prune.** Periodic (or contradiction-triggered) scans for contradictions, orphan pages, stale claims, and missing concept pages. Pages that fail lint get `structural_confidence` adjusted downward, immediately changing retrieval weight. Stale pages trigger re-ingestion and recompilation.

**Retrieval analytics (new in CWAR).** Aggregated retrieval patterns — which pages are consistently retrieved together, which queries return low-confidence results, which topics have high query volume but sparse coverage — feed back into compilation priority. The compiler focuses effort where the knowledge base is weakest relative to demand.

---

## On error propagation

If the LLM compiles an erroneous page, subsequent compilations that cite it can propagate the error. At small scale, you catch mistakes manually. At large scale, this breaks down.

CWAR can't eliminate LLM fallibility. But it addresses error propagation at multiple layers:

**Scoring-level demotion.** Structural confidence naturally penalizes error-prone pages: single-source pages carry lower `source_count_score`, pages with detected contradictions take a -0.2 penalty, pages that age without re-lint lose freshness weight. Errors in low-confidence pages are far less likely to surface in responses.

**Cross-model lint.** The compilation LLM and the lint LLM share blind spots when they're the same model. Running lint with a different model (e.g., compile with Claude, lint with GPT-4) introduces independent verification. This doesn't catch all errors, but it breaks the correlation between compilation errors and lint misses.

**Human spot-check workflow.** Periodic random sampling of high-influence pages (high inbound cross-references, high retrieval count) for manual review. A `human_review_flag` in frontmatter provides a +0.1 structural confidence boost and signals that the page has been verified by a human at least once.

The combination doesn't prevent errors. It makes them visible, reduces their retrieval impact, and creates recovery paths.

---

## Getting started

If you're already running an LLM Wiki, CWAR is an additive extension. If starting from scratch, start with LLM Wiki and add the vector layer when you need it.

**Weeks 1–2: Pure LLM Wiki.** Build `raw/`, `wiki/`, `index.md`, `log.md`. Establish your schema. **Add YAML frontmatter from day one** — it costs nothing now and enables everything later. Get compilation quality right: good cross-references, consistent categories, disciplined `Data Gaps` sections. No amount of retrieval engineering compensates for a sloppy wiki.

**Weeks 3–4: Wire in the vector DB (~100 pages).** pgvector is the right starting point — PostgreSQL extension, free, runs on existing infrastructure. Index frontmatter as scalar columns; embed TL;DR, sections, and `questions_answerable` as separate vectors; store cross-references as adjacency lists for graph traversal; implement hybrid search with metadata filtering.

**Weeks 5–8: Automate.** Git hook or file watcher on `raw/` triggering compile → lint → embed. Retrieval hit count tracking. Source type detection for adaptive compilation. This is when the feedback loops start running on their own.

**Weeks 9–12: Close all loops.** Automatic filing of query responses back to the wiki. Periodic lint-prune passes. Retrieval analytics dashboard: average `structural_confidence` over time, cross-reference density, retrieval hit distribution, query-to-coverage gaps.

**Technology choices.** pgvector for small-to-medium, Weaviate for larger deployments, Milvus for serious scale. `text-embedding-3-small` for general corpora; Voyage AI for code-heavy documentation. Cohere Rerank or a local cross-encoder for reranking. Claude Opus for page creation and cross-document integrations, Claude Haiku for lint. With prompt caching, monthly costs for a 300-page wiki run roughly $7–42 depending on ingest volume.

---

## What CWAR is not

**Not a real-time system.** There's latency between ingestion and index availability. For immediately retrievable documents, maintain a parallel vanilla RAG path over raw sources as a fallback.

**Not a substitute for wiki discipline.** The retrieval index is downstream of compilation quality. Weak cross-references, missing Data Gaps, unstable confidence scores — all propagate to degraded retrieval.

**Not limited to single-agent use, but single-agent is the default.** Multi-agent concurrent compilation requires coordination. The practical solution: a compilation queue (FIFO) with per-page locks. Compilations are serialized per page; lint passes can run in parallel (no conflict risk). This avoids CRDT complexity while enabling team use. Git branch-per-agent with merge arbitration remains an option for heavier concurrent workloads.

**Not yet empirically benchmarked.** Which leads to the next section.

---

## Benchmark design

CWAR's scoring improvements are reasoned extrapolations from related work — Contextual Retrieval, Chunk Enrichment, LLM Wiki community reports. CWAR-specific empirical validation is needed.

**Proposed experiment:**

- **Corpus:** 50–100 technical documents (API docs, incident reports, design documents — mixed types)
- **Question set:** 30 questions — 10 factual lookup, 10 cross-cutting comparison, 10 causal reasoning
- **Comparison systems:**
  - (A) Vanilla RAG over raw chunks
  - (B) Contextual Retrieval (Anthropic's approach) over raw chunks
  - (C) CWAR (compiled pages + structural confidence scoring)
  - (D) CWAR + lightweight graph traversal

**Metrics:**

| Metric | What it measures |
|--------|-----------------|
| Precision@5 | Relevance of top results |
| Recall@5 | Coverage of relevant pages |
| MRR | How quickly the right page appears |
| Indexing cost (tokens, $) | Build-time expense |
| Query latency (p50, p95) | Runtime performance |

**Key hypothesis:** If CWAR's improvement over Contextual Retrieval exceeds Contextual Retrieval's improvement over vanilla RAG, then compiled knowledge units provide value beyond contextual enrichment of raw chunks. If (D) outperforms (C) significantly on cross-cutting questions specifically, the lightweight graph layer justifies its complexity.

Even at 50 pages, this experiment is feasible and would provide the first empirical grounding for the architecture. If you run it, please share.

---

## Why this works

The expensive part of maintaining a knowledge base is not the reading or thinking, but the bookkeeping — keeping cross-references consistent, updating confidence signals when contradictions appear, propagating changes from new sources to dependent pages. In CWAR, this maintenance work *is* the work that improves the retrieval index. You're not paying a tax to keep a retrieval system healthy; you're doing the wiki maintenance you'd be doing anyway, and the retrieval system improves as a side effect.

The human curates sources, directs analysis, asks the right questions. The LLM handles compilation, cross-referencing, and bookkeeping. The vector DB handles serving and scaling. Each layer does what it's actually good at.

---

## Possible extensions

**Fine-tuning pipeline.** A mature CWAR wiki is a high-quality labeled dataset. Pages with `structural_confidence > 0.9` and `source_count > 5` are effectively gold-standard knowledge artifacts. At sufficient coverage, these could feed a fine-tuning pipeline — producing a domain-specialized model that carries the knowledge without retrieval infrastructure.

**Full graph upgrade.** The lightweight adjacency-list approach handles most cross-reference traversal. For domains with deeply interconnected knowledge (medical ontologies, legal precedent chains), upgrading to Neo4j with Leiden community detection may justify the infrastructure cost — essentially merging CWAR and GraphRAG.

**Collaborative compilation protocol.** Beyond the per-page lock queue, a more sophisticated protocol: agents claim topic areas, compile independently within their area, and a merge agent reconciles cross-area references. This maps naturally to how human teams divide knowledge domains.

---

## Note

Like Karpathy's original, this document describes a pattern, not an implementation. The directory structure, the exact frontmatter schema, the weight functions — all depend on your domain and usage patterns. What's non-negotiable: frontmatter on wiki pages from day one, compilation before embedding, structural confidence flowing into retrieval weights, and cross-references serving double duty as graph edges. Everything else is configuration.

The wiki is still a git repo of markdown files. You still get version history, branching, and diffs for free. Obsidian is still the IDE. The LLM is still the programmer. The wiki is still the codebase. CWAR adds a memory layer that scales without a context window ceiling — and that gets better the more you use it.

The only part to resist: thinking of the vector DB as the interesting part. It's plumbing. The interesting part is still `wiki/`.

---

## References

1. Karpathy, A. (2026). *LLM Wiki.* https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f
2. Anthropic. (2024). *Contextual Retrieval.* https://www.anthropic.com/news/contextual-retrieval
3. Microsoft. (2025). *Develop a RAG Solution — Chunk Enrichment Phase.* https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/rag/rag-enrichment-phase
4. Microsoft. (2024). *GraphRAG: From Local to Global.* https://microsoft.github.io/graphrag/
5. RAGFlow. (2025). *From RAG to Context — A 2025 Year-End Review.* https://ragflow.io/blog/rag-review-2025-from-rag-to-context
6. arXiv. (2025). *Metadata-Driven Retrieval-Augmented Generation for Financial Question Answering.* https://arxiv.org/pdf/2510.24402
7. SamurAIGPT. (2026). *llm-wiki-agent.* https://github.com/SamurAIGPT/llm-wiki-agent
8. tobi. (2026). *qmd — local search engine for markdown.* https://github.com/tobi/qmd
9. Shaik, M. H. (2026). *Beyond Fixed Chunks: Semantic Chunking and Metadata Enrichment.* https://medium.com/@shaikmohdhuz/beyond-fixed-chunks-how-semantic-chunking-and-metadata-enrichment-transform-rag-accuracy-07136e8cf562
10. n1n.ai. (2026). *Scaling Personal Knowledge Beyond RAG with Karpathy's LLM Wiki Pattern.* https://explore.n1n.ai/blog/scaling-personal-knowledge-beyond-rag-karpathy-llm-wiki-2026-04-09
