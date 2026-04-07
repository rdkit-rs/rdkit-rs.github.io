---
title: Resources
weight: 4
geekdocNav: true
geekdocAnchor: true
---

## Presentations

Javier Pineda, PhD., presented on Cheminee at a Scientist Show And Tell: [view the slides](/javier-show-and-tell-2023-truncated.html).

## Why Rust?

Rust is a systems programming language offering memory safety without garbage collection, an integrated build system and package manager (cargo), and a strong static type system. For cheminformatics this means:

- **No GC pauses** — consistent latency when searching millions of molecules
- **Memory safety** — no segfaults, no use-after-free, even when wrapping C++ libraries like RDKit
- **Fearless concurrency** — parallelize indexing and search across cores without data races
- **Single binary deployment** — ship a statically linked binary or Docker image with no runtime dependencies

Learn more about Rust in [The Rust Programming Language](https://doc.rust-lang.org/book/).

## Repositories

| Repository | Description |
|-----------|-------------|
| [rdkit-rs/rdkit](https://github.com/rdkit-rs/rdkit) | High-level Rust bindings for RDKit |
| [rdkit-rs/cheminee](https://github.com/rdkit-rs/cheminee) | Chemical structure search engine |
| [rdkit-rs/cheminee-ruby](https://github.com/rdkit-rs/cheminee-ruby) | Ruby client gem for the Cheminee API |
| [rdkit-rs/cheminee-similarity-model](https://github.com/rdkit-rs/cheminee-similarity-model) | Neural network model for similarity search clustering |
| [rdkit-rs/rdkit-rs.github.io](https://github.com/rdkit-rs/rdkit-rs.github.io) | This website |

## Issues

Please [file an issue on GitHub](https://github.com/rdkit-rs/rdkit/issues) for rdkit-rs, or [here for Cheminee](https://github.com/rdkit-rs/cheminee/issues).
