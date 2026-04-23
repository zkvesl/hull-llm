# Quick Start

For evaluating hull-llm or deploying the verified-RAG pipeline.

**Prerequisites:** [Nockchain](https://github.com/zorp-corp/nockchain) monorepo cloned and built at a sibling path, with `hoonc` and `nockchain` in your PATH. Rust nightly (pinned in `rust-toolchain`).

```bash
git clone https://github.com/zkVesl/hull-llm.git
cd hull-llm
cargo build --release
./scripts/demo.sh --no-chain    # full pipeline, no chain needed
```

The demo runs the full pipeline: ingest documents, retrieve chunks, verify in the Hoon kernel, settle locally.

## Settlement modes

hull-llm supports three settlement modes. Set via `--settlement-mode`, `VESL_SETTLEMENT_MODE`, or `settlement_mode` in `vesl.toml`.

| Mode | What happens | Chain required |
|------|-------------|----------------|
| `local` | Kernel verifies, no chain interaction. Default. | No |
| `fakenet` | Full pipeline — sign, build tx, submit to a local nockchain fakenet. | Yes (local) |
| `dumbnet` | Same as fakenet but uses a real seed phrase for key derivation. | Yes (live) |

Precedence: CLI flag > environment variable > `vesl.toml` > mode defaults. Passing `--chain-endpoint` or `--submit` without an explicit mode infers `fakenet`.

## Fakenet walkthrough

Run the full pipeline: ingest documents, retrieve against a query, verify in the Hoon kernel, build a settlement transaction, sign it, and submit to a local chain.

```bash
# 1. Build
cargo build --release

# 2. Boot a local fakenet (hub + miner, background)
./scripts/fakenet-harness.sh start

# 3. Run the demo with live settlement
./scripts/demo.sh --fakenet

# 4. Or drive it manually via the HTTP API
./target/release/hull-llm --new --serve --settlement-mode fakenet

# In another terminal:
curl -X POST http://127.0.0.1:3000/ingest \
  -H 'Content-Type: application/json' \
  -d '{"documents": ["Q3 revenue: $47M, up 12% YoY"]}'

curl -X POST http://127.0.0.1:3000/query \
  -H 'Content-Type: application/json' \
  -d '{"query": "Summarize Q3 financial position", "top_k": 2}'

# /query triggers: retrieve → LLM → manifest → kernel verify → sign → settle

# 5. Tear down
./scripts/fakenet-harness.sh stop
```

Or do it all in one shot:

```bash
./scripts/fakenet-harness.sh run        # boot → test → teardown
```

The harness mines to a demo signing key so the hull can spend coinbase UTXOs without wallet setup.

## Next steps

- [Configuration](configuration.md) — settlement modes and `vesl.toml` fields
- [CLI Reference](cli-reference.md) — all flags and HTTP endpoints
- [Architecture](architecture.md) — what's inside the hull
- [zkvesl/vesl](https://github.com/zkvesl/vesl) — the underlying SDK and protocol
