---
title: RDKit-rs
geekdocNav: true
# geekdocAlign: center
geekdocAnchor: false
---

The power and speed of RDKit, the safety of Rust! A combination of low level C++ bindings and useful high level Rust
constructs so you can

 * Parse mol/molblocks
 * Normalize
 * Fingerprint

Example:

```rust
use rdkit::{Properties, ROMol};

pub fn main() {
    let mol = ROMol::from_smile("c1ccccc1C(=O)NC").unwrap();
    let properties = Properties::new();
    let computed: HashMap<String, f64> = properties.compute_properties(&mol);
    assert_eq!(*computed.get("NumAtoms").unwrap(), 19.0);
}
```