---
title: Cheminee
weight: 2
geekdocNav: true
geekdocAnchor: true
---

[![Crates.io](https://img.shields.io/crates/v/cheminee.svg)](https://crates.io/crates/cheminee)
[![License](https://img.shields.io/crates/l/cheminee.svg)](https://crates.io/crates/cheminee)

Cheminee is a chemical structure search engine. Index chemical structures with arbitrary metadata, then search by substructure, superstructure, exact match, similarity, or descriptor queries. Built on [Tantivy](https://github.com/quickwit-oss/tantivy) and [rdkit-rs](/docs/rdkit-rs/).

Your callers don't need RDKit — just talk to the REST API.

## Key Features

- **Structure search** — Substructure, superstructure, identity (exact match), and Tanimoto similarity search
- **Descriptor search** — Query by any RDKit descriptor (exactmw, NumAtoms, etc.) or custom metadata
- **SMILES standardization** — Fragment parent, uncharger, and canonicalization in bulk
- **Format conversion** — SMILES to molblock and molblock to SMILES
- **Neural similarity search** — Similarity queries use a neural network encoder ([cheminee-similarity-model](https://github.com/rdkit-rs/cheminee-similarity-model)) to embed Morgan fingerprints into a latent space, then search ranked clusters instead of brute-forcing every compound
- **REST API** — OpenAPI-documented endpoints with built-in Swagger UI
- **CLI** — Batch index SDF files, run queries from the terminal
- **Docker image** — `ghcr.io/rdkit-rs/cheminee`
- **Ruby client** — [cheminee-ruby](https://github.com/rdkit-rs/cheminee-ruby) gem for programmatic access

## API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| POST | `/v1/standardize` | Standardize a list of SMILES |
| POST | `/v1/convert/mol_block_to_smiles` | Convert molblocks to SMILES |
| POST | `/v1/convert/smiles_to_mol_block` | Convert SMILES to molblocks |
| GET | `/v1/schemas` | List available index schemas |
| GET | `/v1/indexes` | List indexes |
| GET | `/v1/indexes/{index}` | Get index details |
| POST | `/v1/indexes/{index}` | Create an index |
| DELETE | `/v1/indexes/{index}` | Delete an index |
| POST | `/v1/indexes/{index}/merge` | Merge index segments |
| POST | `/v1/indexes/{index}/bulk_index` | Index SMILES with metadata |
| DELETE | `/v1/indexes/{index}/bulk_delete` | Delete compounds by SMILES |
| GET | `/v1/indexes/{index}/search/basic` | Basic descriptor/metadata search |
| GET | `/v1/indexes/{index}/search/substructure` | Substructure search |
| GET | `/v1/indexes/{index}/search/superstructure` | Superstructure search |
| GET | `/v1/indexes/{index}/search/identity` | Exact structure match |

## Quick Start with Docker

Run Cheminee:

```bash
docker run --rm -p 4001:4001 ghcr.io/rdkit-rs/cheminee:latest
```

Visit [localhost:4001](http://localhost:4001) for the Swagger UI.

### Index Some Data

Fetch PubChem SDF files and index them:

```bash
docker exec -it cheminee bash

mkdir -p tmp/sdfs
cheminee fetch-pubchem -d tmp/sdfs
cheminee create-index -i tmp/cheminee/index0 -n descriptor_v1 -s exactmw
cheminee index-sdf -s tmp/sdfs/Compound_000000001_000500000.sdf.gz -i tmp/cheminee/index0
```

### CLI Examples

**Basic search** — query by descriptor ranges:

```bash
cheminee basic-search -i /tmp/cheminee/index0 \
  -q "exactmw: [10 TO 10000] AND NumAtoms: [8 TO 100]" -l 10
```

**Substructure search:**

```bash
cheminee substructure-search -i /tmp/cheminee/index0 \
  -s CCC -r 10 -t 10 -u true -e "exactmw: [20 TO 200]"
```

**Similarity search:**

```bash
cheminee similarity-search -i /tmp/cheminee/index0 \
  -s c1ccccc1CC -r 10 -t 10 -p 0.1 -m 0.4
```

## Building from Source

```bash
cargo run --release --package cheminee --bin cheminee -- rest-api-server
```

## Links

- [GitHub: rdkit-rs/cheminee](https://github.com/rdkit-rs/cheminee)
- [crates.io: cheminee](https://crates.io/crates/cheminee)
- [Docker image](https://github.com/rdkit-rs/cheminee/pkgs/container/cheminee)
- [OpenAPI spec](https://github.com/rdkit-rs/cheminee/blob/main/openapi.json)
- [cheminee-ruby](https://github.com/rdkit-rs/cheminee-ruby)
- [cheminee-similarity-model](https://github.com/rdkit-rs/cheminee-similarity-model)
