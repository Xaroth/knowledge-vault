---
type: Design Document
title: "Indexing & Search — Scaling Beyond Tags"
description: "The decision on indexing large and multiple knowledge vaults for fast agent search: a persistent pure-Go BM25 + frontmatter-filter index on SQLite FTS5, tags kept as filters, optional local static-embedding vectors as a later phase. With options compared and cited sources."
tags: [design, speki, indexing, search, okf, tooling, decisions]
timestamp: 2026-07-26T12:00:00Z
---

# 4. Indexing & Search — Scaling Beyond Tags

**Decision:** add a **persistent BM25 full-text index with frontmatter-filter columns, built on
`modernc.org/sqlite` FTS5** (pure-Go, single file, zero service, gitignored, content-hash
incremental). **Keep tags as a first-class filter dimension**, not the primary ranker. Treat **local
static-embedding vector search as an optional, later phase** behind a clean seam — worth it only for
vocabulary-gap and "find related" queries.

Bottom line up front from the research: for an agent that can already grep and read files, **a
persistent BM25 index + tag filters covers ~90% of the value with zero service.** Full HNSW/vector
machinery is likely over-engineering at vault scale; if semantics are added, static embeddings +
brute-force cosine are the zero-service way to do it.

## 4.1 The problem restated

Today's discovery (§1.6) is **tags only**, computed on the fly with **no persisted index** and **no
body search** (`--query` matches title/description substrings; full-text is punted to ripgrep). The
stated worry is that a single flat tag vocabulary **drifts and dilutes across large and multiple
vaults** and will not survive the test of time. The fix is not "better tags" — it is **adding a ranked
body index and demoting tags to a filter**, so tags stop being asked to do a search engine's job.

## 4.2 Full-text options (embedded, zero-service)

### SQLite FTS5 via `modernc.org/sqlite` — recommended

FTS5 gives an inverted index with **built-in BM25 ranking**, the **porter** stemmer, prefix and
proximity (`NEAR`) queries, and sub-millisecond retrieval
([SQLite FTS5 docs](https://www.sqlite.org/fts5.html)). The decisive fact for speki's constraint:
**`modernc.org/sqlite` is a CGo-free transpiled port** of SQLite with FTS5 compiled into the standard
amalgamation — so BM25 full-text with **no C toolchain**, preserving trivial cross-compilation and
`self-update` ([modernc.org/sqlite](https://pkg.go.dev/modernc.org/sqlite)).

Why it fits best:

- **One file, tiny footprint, trivial incremental upserts.**
- **SQL joins full-text and frontmatter in one query** — filter by tag/type/path columns *and* rank
  by BM25 in a single statement. This is exactly the "tags become filters, BM25 becomes the ranker"
  shape.
- **A clean upgrade seam:** `sqlite-vec` vectors can later live in the *same* database file (§4.3).

Cost: less linguistic flexibility than Bleve's analyzer chain, and pure-Go modernc is slower than CGo
SQLite (irrelevant at vault scale). Note a constraint that shapes the vector decision: **modernc
cannot load external SQLite loadable extensions**, so `sqlite-vec` would require the CGo driver — a
reason to keep vectors in a separate Go-native store instead.

### Bleve — the single-library alternative

`github.com/blevesearch/bleve/v2` is a mature (~11k★, Couchbase-backed), **pure-Go, Apache-2.0**
full-text engine with a configurable analysis pipeline, 30+ language analyzers, and — notably — **built
-in approximate-KNN vector search plus RRF/RSF fusion**, so full-text + vector + hybrid in one package
([README](https://github.com/blevesearch/bleve)). The trade-off vs SQLite: a **larger on-disk index**
(posting lists + positions + stored fields), background segment merging, and a heavier non-SQL API.
Choose Bleve only if a one-library hybrid stack is preferred over SQLite's smaller, SQL-joinable
single file. For speki's "add ranked body search with minimum surface area," **FTS5 wins**.

### Trigram / ripgrep — keep as the exact-match escape hatch

- **ripgrep** (current fallback): no index, always fresh, great for exact/regex, but no ranking or
  stemming and O(corpus) per query. Keep it as the agent's precise-match tool; it is a poor primary
  *ranked* search.
- **`github.com/google/codesearch`** (BSD-3, Russ Cox): a trigram inverted index for **fast indexed
  regex/substring** over huge trees ([README](https://github.com/google/codesearch)). Optional, only
  if indexed regex over very large trees becomes a hot path. It gives "grep but instant," not ranking
  or semantics.

## 4.3 Vector / semantic search — optional, zero-service if adopted

Only pursued if BM25 + tags prove insufficient for vocabulary-gap queries (§4.6). If adopted, it must
still be zero-service:

- **Local embeddings without a daemon — static embeddings (model2vec-style).** model2vec distills a
  sentence-transformer into a **static token→vector table**; inference is "look up each token's vector
  and mean-pool" — no attention, no matmul — at **~92% of MiniLM MTEB quality** in a ~30 MB model, up
  to ~500× faster on CPU ([model2vec](https://github.com/MinishLab/model2vec)). There is no official
  Go port, but the inference is trivial enough to **reimplement in near-pure Go** (tokenizer +
  safetensors lookup + mean-pool); the existence of an official Rust port confirms it needs no ONNX
  runtime. **This is the best "semantics with zero service and near-pure-Go" candidate.**
  - The full-quality alternative, **ONNX Runtime (`yalue/onnxruntime_go`) running all-MiniLM-L6-v2**,
    works but is **CGo / loads a native shared library** plus a ~90 MB model — against the pure-Go
    preference. **Ollama/LocalAI are ruled out**: they are separate daemons, violating zero-service.
- **Local vector storage — `chromem-go`** (`philippgille/chromem-go`): **pure-Go, zero-dependency**,
  MPL-2.0, brute-force exact cosine, in-memory with optional persistence. Published latency
  **~0.5 ms @ 1k docs, ~40 ms @ 100k docs** ([README](https://github.com/philippgille/chromem-go)).
  `sqlite-vec` is the SQL-native alternative but needs CGo (and can't load into modernc), and HNSW
  engines (`usearch`) only pay off past ~100k–1M vectors.
- **Brute force is genuinely fine at vault scale.** 10k chunks × 384 dims × 4 bytes ≈ **~15 MB** RAM;
  a linear cosine scan is a few ms. **No HNSW needed below ~100k vectors.**

## 4.4 Hybrid search (BM25 + vector, RRF)

If vectors are added, fuse them with BM25 via **Reciprocal Rank Fusion** — it fuses on *ranks* not
scores (sidestepping scale-incompatibility) and is ~15 lines: take top-N from FTS5 and top-N from
cosine, score each doc `Σ 1/(k+rank)` (k≈60), sort. Evidence shows hybrid reliably beats either alone
(e.g. ~+7.4% NDCG on WANDS; recall@10 jumps reported from ~65–78% to ~91%)
([InfoQ](https://www.infoq.com/articles/vector-search-hybrid-retrieval-rag/)). Bleve ships RRF/RSF
natively if that engine is chosen instead. Cheap to add — but only *if* vectors are added at all.

## 4.5 Chunking, metadata, and index lifecycle

- **Chunk by Markdown heading hierarchy** (`#`/`##`/`###`), propagating the header path as metadata,
  then sub-chunk oversized sections to ~256–512 tokens with light overlap; keep code blocks/tables
  intact ([Firecrawl chunking guide](https://www.firecrawl.dev/blog/best-chunking-strategies-rag)).
  This suits both BM25 (section-scoped hits) and future vectors.
- **Frontmatter → structured filter columns** stored per chunk (tags, type, title, path, section,
  mtime, content-hash). **Filter first (tags/path/type), then rank (BM25/vector) within the filtered
  set** — this is exactly where the existing tag index keeps earning its keep, joined to a body index
  rather than replaced. Tag normalization must reuse `NormalizeTag` so the index agrees with the
  validator/catalog.
- **Incremental freshness by content hash, not mtime** (mtime resets on clone; the git blob SHA is
  stable). Store `(path, hash, indexed_at)`; on each run, re-index only changed/new/deleted files —
  retiring the current "rebuild every run" cost. In a git repo, deltas can also come from
  `git diff --name-only`.
- **The index is a local, gitignored cache** (e.g. `.speki/index/` or an XDG cache dir), **never
  committed.** It is large, binary, and merge-hostile; committing it would bloat history and conflict
  constantly across multiple vaults. Rebuild-on-demand is cheap and deterministic — the same way
  code-search tools treat indexes as disposable artifacts.

## 4.6 "Do you even need vectors?" — the honest counterpoint

There is strong real-world signal that **for an agent, good full-text + tags is often enough**.
Anthropic dropped RAG/vector search in Claude Code in favour of **agentic grep/glob/read**, reporting
it "outperformed everything, by a lot," because the model iteratively refines queries, follows
references, and self-corrects — with **no index lag** ([Claude Code doesn't
index](https://vadim.blog/claude-code-no-indexing/)). Vectors earn their complexity mainly for
**vocabulary-gap** queries (the exact words aren't present — "climate policy" matching a doc that
never uses those words) and **cross-vault "find related"** discovery, where BM25 can't bridge synonyms
but embeddings can.

The nuance that still justifies the BM25 index: Claude Code's "just grep" works because the *agent*
drives the loop live. A speki *tool* that exposes a single ranked `search` call benefits from ranking
the agent cannot cheaply reproduce — so **a persistent BM25 index is a clear win over raw ripgrep**,
while **vectors remain optional polish** gated on real need. This is why the plan lands BM25 now and
leaves a seam for vectors, rather than reaching for the vector stack first.

## 4.7 Recommendation (ranked)

1. **Persistent BM25 body index + tag/frontmatter filter columns on `modernc.org/sqlite` FTS5.**
   Pure-Go, single file, zero service, content-hash incremental, tags kept as filters. Highest-value,
   lowest-risk fix for "rebuild every run / no body search / tags don't scale." (License: SQLite
   public domain + modernc BSD-3.)
2. **Keep ripgrep** as the exact/regex escape hatch; add **google/codesearch** trigram only if indexed
   regex over huge trees becomes hot.
3. **Optional later phase — local semantics with zero service:** static embeddings (model2vec-style,
   near-pure-Go: token lookup + mean-pool) + brute-force cosine (or chromem-go), fused with BM25 via
   ~15-line RRF. Brute force is fine to ~100k chunks (~40 ms).
4. **Avoid under the constraint:** Ollama/LocalAI (daemon); ONNX/usearch/sqlite-vec-CGo (native deps)
   unless CGo is explicitly accepted; HNSW only past ~100k–1M vectors.
5. **Single-library alternative:** Bleve, if a one-package full-text + vector + RRF stack is preferred
   over SQLite's smaller, SQL-joinable file.
6. **Index lifecycle:** local, gitignored, content-hash incremental — never committed across vaults.

**Key sources:** [SQLite FTS5](https://www.sqlite.org/fts5.html) ·
[modernc.org/sqlite](https://pkg.go.dev/modernc.org/sqlite) ·
[chromem-go benchmarks](https://github.com/philippgille/chromem-go) ·
[model2vec](https://github.com/MinishLab/model2vec) ·
[Bleve](https://github.com/blevesearch/bleve) · [google/codesearch](https://github.com/google/codesearch) ·
[Claude Code — no index / agentic search](https://vadim.blog/claude-code-no-indexing/) ·
[hybrid search / RRF (InfoQ)](https://www.infoq.com/articles/vector-search-hybrid-retrieval-rag/)
