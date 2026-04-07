---
title: rdkit-rs
weight: 1
geekdocNav: true
geekdocAnchor: true
---

[![Crates.io](https://img.shields.io/crates/v/rdkit.svg)](https://crates.io/crates/rdkit)
[![License](https://img.shields.io/crates/l/rdkit.svg)](https://crates.io/crates/rdkit)

Safe, idiomatic Rust bindings for [RDKit](https://www.rdkit.org/), the industry-standard open-source cheminformatics library.

## What It Provides

The project ships two crates:

- **`rdkit-sys`** — Low-level C++ bindings via [cxx](https://crates.io/crates/cxx). Zero-cost wrappers exposing a key subset of RDKit's C++ API.
- **`rdkit`** — High-level Rust library built on `rdkit-sys`. No manual memory management, no null pointers. Implements `Debug`, `Clone`, and idiomatic borrowing so molecules behave like native Rust types.

## Capabilities

| Area | What You Can Do |
|------|----------------|
| **Parsing** | SMILES, molblocks, SDF files (including gzipped) |
| **Normalization** | Fragment parent, uncharger, canonical tautomer |
| **Fingerprints** | Morgan fingerprints, pattern fingerprints |
| **Descriptors** | Compute all standard RDKit descriptors (exactmw, NumAtoms, CrippenClogP, etc.) |
| **Tautomers** | Enumerate tautomers, canonicalize |
| **Substructure** | SMARTS-based substructure and superstructure matching |
| **Periodic Table** | Element lookups and properties |

## Quick Start

Add to your `Cargo.toml`:

```toml
[dependencies]
rdkit = "0.4"
```

Example:

```rust
use rdkit::{Properties, ROMol};
use std::collections::HashMap;

fn main() {
    let mol = ROMol::from_smile("c1ccccc1C(=O)NC").unwrap();
    let properties = Properties::new();
    let computed: HashMap<String, f64> = properties.compute_properties(&mol);
    assert_eq!(*computed.get("NumAtoms").unwrap(), 19.0);
}
```

Browse more examples in the [examples directory](https://github.com/rdkit-rs/rdkit/tree/main/examples).

## Prerequisites

Requires RDKit 2023.09.1 or higher.

**macOS:**

```bash
brew install rdkit
```

**Linux (Ubuntu 24.04+):**

Pre-compiled static library tarballs are available for AMD64 and ARM64:

- [AMD64](https://rdkit-rs-debian.s3.eu-central-1.amazonaws.com/rdkit_2024_03_3_ubuntu_14_04_amd64.tar.gz)
- [ARM64](https://rdkit-rs-debian.s3.eu-central-1.amazonaws.com/rdkit_2024_03_3_ubuntu_14_04_arm64.tar.gz)

You will also need a C++ compiler (we recommend clang) for building the `rdkit-sys` bridge code.

## Rust Version

We support recent stable Rust versions. The limiting factor is [cxx](https://crates.io/crates/cxx) — check the [cxx Cargo.toml](https://github.com/dtolnay/cxx/blob/master/Cargo.toml#L6) for the minimum `rust-version`.

## Links

- [GitHub: rdkit-rs/rdkit](https://github.com/rdkit-rs/rdkit)
- [crates.io: rdkit](https://crates.io/crates/rdkit)
- [crates.io: rdkit-sys](https://crates.io/crates/rdkit-sys)
