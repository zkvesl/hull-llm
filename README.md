# hull-llm

Verifiable RAG on Nockchain: ingest documents, retrieve chunks, run an LLM, prove the pipeline, settle on-chain. A reference Hull built on `vesl-core`.

Extracted from [zkvesl/vesl](https://github.com/zkvesl/vesl) as a standalone repo. The generic Hull template and protocol live in `vesl`; everything LLM-flavored — Ollama integration, retrieval backends, ingest pipeline, document corpus — lives here.

## Build

```
cargo build --release
cargo build --release --features dumbnet   # real-key settlement via wallet kernel
```

## Run

```
./target/release/hull-llm --new --serve       # HTTP API
./target/release/hull-llm --new --query "..."  # one-shot
```

Configure via `vesl.toml` (see `docs/configuration.md`).

## Layout

- `src/` — hull-llm binary + library
- `tests/` — e2e integration tests (fakenet, adversarial, prover, ollama)
- `kernels/vesl/` — the compiled Vesl kernel JAM wrapper (Hoon source lives upstream in zkvesl/vesl)
- `kernels/forge/` — STARK prover kernel wrapper (dev/test only)
- `assets/` — vesl.jam and forge.jam binaries + SHA checksums
- `demo/docs/` — sample corpora for RAG demo
- `scripts/` — demo.sh, fakenet-harness.sh, diag-reproduce.sh
