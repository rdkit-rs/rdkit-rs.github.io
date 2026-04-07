---
title: Roadmap
weight: 3
geekdocNav: true
geekdocAnchor: true
---

## Quickwit Integration

The next major milestone for Cheminee is integration with [Quickwit](https://quickwit.io), a cloud-native search engine built on Tantivy. This will bring S3-backed storage, elastic compute scaling, and the full Quickwit operational model to chemical structure search.

### Why Quickwit?

Cheminee currently runs as a standalone server wrapping Tantivy directly. This works well, but requires a dedicated server sized for peak indexing workloads even when most of the time is spent serving search traffic. Quickwit solves this by separating compute from storage:

- **Index splits go to S3** — no local disk to manage
- **Indexers scale independently from searchers** — burst to many cores for indexing, scale to zero when idle
- **Searchers stay small** — they pull splits from S3 on demand

### Architecture

```
                    +-----------------------+
                    |   quickwit-cheminee   |  <-- new crate, the glue
                    +-----------------------+
                       /        |        \
          +-------------+   +-------------+   +------------+
          | DocProcessor|   |  Tokenizer  |   |  Query +   |
          | (transform) |   |  (indexing) |   | Collector  |
          +-------------+   +-------------+   | (search)   |
               |               |              +------------+
          +----------+   +-----------+              |
          |  rdkit   |   |  rdkit    |         +----------+
          +----------+   +-----------+         |  rdkit   |
                                               +----------+
```

A `quickwit-cheminee` crate will plug into Quickwit at compile time via a cargo feature flag, adding chemistry awareness at three points:

1. **Indexing** — A custom `DocProcessor` that takes raw SMILES, standardizes them, computes fingerprints and descriptors, and enriches the document before it enters the index
2. **Tokenization** — A `SmilesTokenizer` for chemistry-aware text indexing
3. **Search** — Custom query types (substructure, superstructure, similarity, identity) and collectors that perform chemical post-filtering using fingerprint screening and RDKit substructure matching

### Key Design Decisions

- **`cheminee-core` extraction** — Chemistry logic (standardization, fingerprinting, descriptor computation, substructure matching) will be extracted into a standalone `cheminee-core` crate, shared by both the existing Cheminee server and the Quickwit plugin
- **`fingerprint` field type** — A first-class Tantivy/Quickwit field type for chemical fingerprints, enabling fast Tanimoto similarity queries and substructure screening directly in the index
- **Native Quickwit doc mapping** — Use Quickwit's existing schema system rather than maintaining a separate schema library

### Milestones

| Phase | Work | Status |
|-------|------|--------|
| 1 | Extract `cheminee-core` crate | Planned |
| 2 | Implement `ChemicalDocMapper` for indexing | Planned |
| 3 | Implement chemical search queries + collectors | Planned |
| 4 | REST API and integration testing | Planned |
| 5 | Deploy and validate | Planned |

### Target Quickwit Version

We're targeting **Quickwit release-0.9** (`v0.9.0`). Key changes from earlier Quickwit versions that affect our integration:

- `DocMapper` is now a concrete struct (not a `dyn` trait)
- Indexing pipeline: `Source → DocProcessor → Indexer → IndexSerializer → Packager → Uploader → Sequencer → Publisher`
- Tantivy is pinned to a Quickwit fork
- License is Apache 2.0
