---
title: RDKit-rs
geekdocNav: true
# geekdocAlign: center
geekdocAnchor: false
---

<!-- Ranges through content/tutorials/*.md -->
{{ range .Pages }}
    <li>
        <a href="{{.Permalink}}">{{.Date.Format "2006-01-02"}} | {{.Title}}</a>
    </li>
{{ end }}


[![License](https://img.shields.io/crates/l/rdkit.svg)](https://crates.io/crates/rdkit)
[![Crates.io](https://img.shields.io/crates/v/rdkit.svg)](https://crates.io/crates/rdkit)

The power and speed of [RDKit](https://www.rdkit.org/), the safety of Rust! A combination of low level C++ bindings and useful high level Rust
constructs so you can

 * Parse mol/molblocks
 * Normalize
 * Fingerprint
 * Enumerate tautomers/canonicalize

How does it work?
---

The rdkit-rs project provides two key libraries: `rdkit` and `rdkit-sys`. The sys package is a collection of low or zero-cost wrappers exposing a key subset of the RDKit C++ functionality. The `rdkit` package builds on top of the sys package, hiding pointers and providing idiomatic Rust interfaces (think: `Debug` and `Clone` implementations, smart borrowing behavior).

With the `rdkit` library you will never need to manually free memory or worry about accessing null pointers. You also get all the benefits of an optimizing compiler and will never wait for garbage collection.

Example
---

in your Cargo.toml:

```
[dependencies]
rdkit = "*"
```

If you satisfy the requirements below, the following code should just compile!

```rust
use rdkit::{Properties, ROMol};

pub fn main() {
    let mol = ROMol::from_smile("c1ccccc1C(=O)NC").unwrap();
    let properties = Properties::new();
    let computed: HashMap<String, f64> = properties.compute_properties(&mol);
    assert_eq!(*computed.get("NumAtoms").unwrap(), 19.0);
}
```

Browse more [rdkit-rs/rdkit examples](https://github.com/rdkit-rs/rdkit/tree/main/examples)

Requirements
---

We support recent stable Rust versions. The limiting factor is whatever [our C++ bindings library, cxx-rs](https://crates.io/crates/cxx), supports. Check [the cxx Cargo.toml](https://github.com/dtolnay/cxx/blob/master/Cargo.toml#L6) to confirm what `rust-version` is supported.

Requires a recent version of RDKit, tested against `2022.03.1`. Supports both static and dynamic linking, preferring static linking.
You can use a copy of RDKit installed either from [Mac homebrew](https://homebrew.sh) or [Conda Forge](https://anaconda.org/). We are working to
get Debian packages updated for the most recent RDKit and also including static libraries so we can build portable RDKit applications.

 * brew install rdkit
 * conda install -c conda-forge rdkit==2022.03.1

Ubuntu support is coming soon. 

You will also need a compiler for building the sys package's C++ bridge. We recommend clang for the compilation speed.

Why Rust?
---

Rust is a powerful systems level programming language, offering a smart static typing system, an integrated build system and package manager, and strong memory safety, among many other benefits. Read more about Rust in [the free Rust Book](https://doc.rust-lang.org/book/).

Issues?
---

Please [file an issue on GitHub](https://github.com/rdkit-rs/rdkit/issues)
