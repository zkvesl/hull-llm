# Architecture

hull-llm is a Rust harness that boots the Vesl kernel (`vesl.jam`) and wraps it with an ingest / retrieve / LLM / settle pipeline. It sits on top of `vesl-core` for the protocol primitives.

## Responsibilities

- Boot the Vesl Hoon kernel as an embedded NockApp
- Ingest documents into a tip5 Merkle tree
- Retrieve chunks with keyword scoring
- Build and verify manifests (prompt integrity, proof paths)
- LLM inference (Ollama or deterministic stub)
- Schnorr signing and transaction construction
- On-chain settlement via Nockchain gRPC
- Expose REST API (`/ingest`, `/query`, `/prove`, `/status`, `/health`)
- Route between settlement modes (local, fakenet, dumbnet)

## Request flow

```
┌─────────────────────────────────────┐
│  client                             │
└─────────────────┬───────────────────┘
                  │ HTTP
┌─────────────────┴─────────────────────┐
│  api.rs                        axum   │
│  /ingest /query /prove /status /health│
└──┬─────────┬─────────┬────────────────┘
   │         │         │
   ▼         ▼         │
┌──────┐ ┌────────┐    │
│ingest│ │retrieve│    │
│ .rs  │ │  .rs   │    │
└──┬───┘ └───┬────┘    │
   │         ▼         │
   │    ┌────────┐     │
   │    │ llm.rs │     │
   │    └───┬────┘     │
   │        │          │
   ▼        ▼          ▼
┌─────────────────────────────────────┐
│  merkle.rs       tip5 Merkle tree   │
│  noun_builder.rs → kernel poke      │
│  ┌───────────────────────────────┐  │
│  │ vesl.jam      hoon kernel     │  │
│  │ verify manifest + merkle root │  │
│  └───────────────────────────────┘  │
└─────────────────┬───────────────────┘
                  │
   ┌──────────────┼──────────────┐
   ▼              ▼              ▼
┌────────┐ ┌───────────┐ ┌──────────┐
│signing │ │tx_builder │ │ chain.rs │
│  .rs   │ │   .rs     │ │  gRPC    │
│schnorr │ │ assemble  │ │  submit  │
└────────┘ └───────────┘ └──────────┘
   │              │              │
   └──────────────┼──────────────┘
                  ▼
┌─────────────────────────────────────┐
│  nockchain                          │
│  mode: local │ fakenet │ dumbnet    │
└─────────────────────────────────────┘
```

## Key modules

| Module | What it does |
|--------|-------------|
| `merkle.rs` | tip5 Merkle tree, cross-runtime aligned with Hoon |
| `chain.rs` | On-chain settlement + confirmation via gRPC |
| `api.rs` | HTTP API server (axum) with 10 MB body limit |
| `tx_builder.rs` | Settlement transaction construction |
| `signing.rs` | Schnorr signing (returns `Result`, no panics) |
| `ingest.rs` | Document chunking into the tree |
| `llm.rs` | LLM integration (trait-based: Ollama or stub) |
| `noun_builder.rs` | Nock noun construction for kernel pokes |
| `retrieve.rs` | Keyword-based chunk retrieval with scoring |
| `config.rs` | Settlement config resolution (CLI > env > toml > defaults) |

## Project layout

```
src/
  api.rs            Axum HTTP routes
  chain.rs          Nockchain gRPC client for settlement
  config.rs         vesl.toml parsing and config precedence
  ingest.rs         Document ingestion into tip5 Merkle tree
  llm.rs            Ollama LLM integration
  merkle.rs         tip5 Merkle tree operations
  noun_builder.rs   Nock noun construction for kernel pokes
  retrieve.rs       Chunk retrieval with keyword scoring
  signing.rs        Schnorr key derivation and transaction signing
  tx_builder.rs     Nockchain transaction construction
  wallet.rs         Wallet noun builders
  wallet_kernel.rs  In-process wallet kernel (feature-gated)
kernels/
  vesl/             Kernel JAM embedding crate
  forge/            STARK prover kernel wrapper (dev/test)
assets/
  vesl.jam          Pre-compiled kernel (~18 MB)
  forge.jam         STARK prover kernel
```

## Customization points

- **`api.rs`** — add endpoints for your domain
- **`ingest.rs`** — change how documents are chunked and indexed
- **`llm.rs`** — swap Ollama for a different inference provider
- **`retrieve.rs`** — customize retrieval scoring
- **`config.rs`** — add configuration fields to `vesl.toml`

The kernel interaction (`noun_builder.rs`, `merkle.rs`) and settlement pipeline (`chain.rs`, `tx_builder.rs`, `signing.rs`) should rarely need modification.

## Where intents fit

hull-llm serves the commitment layer — family 1 in vesl's 5-family graft catalog. Family 5 (intent coordination: declare / match / cancel / expire) sits *above* commitments and is optional. A NockApp can settle through this Hull without ever declaring an intent. When the Nockchain monorepo publishes a canonical intent structure, vesl swaps the placeholder `intent-graft` for the real primitive; hull-llm endpoints don't need to change. See the [vesl grafting guide](https://github.com/zkvesl/vesl/blob/main/templates/GRAFTING.md) for the full family taxonomy.
