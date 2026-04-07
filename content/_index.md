---
title: RDKit-rs
geekdocNav: true
geekdocAnchor: false
---

[![License](https://img.shields.io/crates/l/rdkit.svg)](https://crates.io/crates/rdkit)
[![rdkit crate](https://img.shields.io/crates/v/rdkit.svg?label=rdkit)](https://crates.io/crates/rdkit)
[![cheminee crate](https://img.shields.io/crates/v/cheminee.svg?label=cheminee)](https://crates.io/crates/cheminee)

**Chemistry infrastructure in Rust** — from molecular parsing to full-text structure search.

The rdkit-rs organization builds safe, fast, open-source tools for cheminformatics on top of [RDKit](https://www.rdkit.org/) and the Rust ecosystem.

---

## Projects

### [rdkit-rs](/docs/rdkit-rs/)

Safe Rust bindings for the RDKit C++ library. Parse SMILES and molblocks, normalize molecules, compute fingerprints, enumerate tautomers, calculate descriptors — all with Rust's memory safety guarantees and zero garbage collection overhead.

```rust
use rdkit::{Properties, ROMol};

let mol = ROMol::from_smile("c1ccccc1C(=O)NC").unwrap();
let properties = Properties::new();
let computed = properties.compute_properties(&mol);
assert_eq!(*computed.get("NumAtoms").unwrap(), 19.0);
```

### [Cheminee](/docs/cheminee/)

A chemical structure search engine built on [Tantivy](https://github.com/quickwit-oss/tantivy) and rdkit-rs. Index millions of molecules and search by substructure, superstructure, exact match, or similarity. Ships as a REST API with Swagger UI, a CLI for batch operations, and a Docker image ready to run.

### [Roadmap: Quickwit Integration](/docs/roadmap/)

We're working to make Cheminee a first-class plugin inside [Quickwit](https://quickwit.io), bringing S3-backed storage, elastic compute scaling, and the full Quickwit operational model to chemical search.

---

## Get Started

Install the Rust crate:

```toml
[dependencies]
rdkit = "0.4"
```

Or run Cheminee in Docker:

```bash
docker run --rm -p 4001:4001 ghcr.io/rdkit-rs/cheminee:latest
```

Then visit [localhost:4001](http://localhost:4001) for the Swagger UI.

---

## Links

- [GitHub Organization](https://github.com/rdkit-rs)
- [rdkit on crates.io](https://crates.io/crates/rdkit)
- [cheminee on crates.io](https://crates.io/crates/cheminee)
- [Resources & Presentations](/docs/resources/)
